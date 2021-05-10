# `MikanLoaderPkg/Main.c`を読む

## 1. エントリ関数: `UefiMain()`=>ブートローダとして働く

```cpp
EFI_STATUS EFIAPI UefiMain(             // 関数名は`MikanLoaderPkg/Loader.inf`の`ENTRY_POINT`で指定する
    EFI_HANDLE image_handle,            // この関数はUEFIが呼び出す: 4.1 UEFI Image Entry Point
    EFI_SYSTEM_TABLE *system_table) {
    EFI_STATUS status;

// 1. メモリマップの処理: UEFIのメモリマップは仮想メモリ＝物理アドレス=>物理アドレスマップとして使用する

// 1.1 メモリマップを取得する
    CHAR8 memmap_buf[4096 * 8];
    struct MemoryMap memmap = {sizeof(memmap_buf), memmap_buf, 0, 0, 0, 0};
    status = GetMemoryMap(&memmap);
    if (EFI_ERROR(status)) {
        Print(L"failed to get memory map: %r\n", status);
        Halt();
    }

// 1.2. メモリマップをファイルに書き出す

// 1.2.1 disk.imgのルートディレクトリを開く
    EFI_FILE_PROTOCOL *root_dir;
    status = OpenRootDir(image_handle, &root_dir);
    if (EFI_ERROR(status)) {
        Print(L"failed to open root directory: %r\n", status);
        Halt();
    }

// 1.2.2 メモリマップを書き出すファイルを開く
    EFI_FILE_PROTOCOL *memmap_file;
    status = root_dir->Open(                                        // 13.5 File Protocol
        root_dir, &memmap_file, L"\\memmap",
        EFI_FILE_MODE_READ | EFI_FILE_MODE_WRITE | EFI_FILE_MODE_CREATE, 0);
    if (EFI_ERROR(status)) {
        Print(L"failed to open file '\\memmap': %r\n", status);
        Print(L"Ignored.\n");
    } else {
// 1.2.3 メモリマップをファイルに書き出す
        status = SaveMemoryMap(&memmap, memmap_file);
        if (EFI_ERROR(status)) {
            Print(L"failed to save memory map: %r\n", status);
            Halt();
        }
// 1.2.4 メモリマップファイルを閉じる
        status = memmap_file->Close(memmap_file);                   // 13.5 File Protocol
        if (EFI_ERROR(status)) {
            Print(L"failed to close memory map: %r\n", status);
            Halt();
        }
    }

// 2. フレームバッファの処理: 画面描画に使用

// 2.1 GRAPHICS_OUTPUT_PROTOCOLを取得する: GOPはディスプレイドライバを抽象化したもの

    EFI_GRAPHICS_OUTPUT_PROTOCOL *gop;
    status = OpenGOP(image_handle, &gop);
    if (EFI_ERROR(status)) {
        Print(L"failed to open GOP: %r\n", status);
        Halt();
    }

// 2.2 画面を白で埋める
    UINT8 *frame_buffer = (UINT8 *)gop->Mode->FrameBufferBase;
    for (UINTN i = 0; i < gop->Mode->FrameBufferSize; ++i) {
        frame_buffer[i] = 255;
    }

// 3. カーネルを読み込み、ELFファイルを処理する

// 3.1 カーネルファイルをバッファに取り込む

// 3.1.1 カーネルファイルを開く
    EFI_FILE_PROTOCOL *kernel_file;
    status = root_dir->Open(
        root_dir, &kernel_file, L"\\kernel.elf",
        EFI_FILE_MODE_READ, 0);
    if (EFI_ERROR(status)) {
        Print(L"failed to open file '\\kernel.elf': %r\n", status);
        Halt();
    }

// 3.1.2 カーネルファイルをバッファに取り込む
    VOID *kernel_buffer;
    status = ReadFile(kernel_file, &kernel_buffer);
    if (EFI_ERROR(status)) {
        Print(L"error: %r\n", status);
        Halt();
    }

// 3.2 ELF形式のカーネルファイルを処理する

// 3.2.1 カーネルサイズを取得する
    Elf64_Ehdr *kernel_ehdr = (Elf64_Ehdr *)kernel_buffer;
    UINT64 kernel_first_addr, kernel_last_addr;
    CalcLoadAddressRange(kernel_ehdr, &kernel_first_addr, &kernel_last_addr);

// 3.2.2 必要なページ数を計算して、メモリを割り当てる
    UINTN num_pages = (kernel_last_addr - kernel_first_addr + 0xfff) / 0x1000;
    status = gBS->AllocatePages(AllocateAddress, EfiLoaderData, num_pages, &kernel_first_addr);
    if (EFI_ERROR(status)) {
        Print(L"failed to allocate pages: %r\n", status);
        Halt();
    }

// 3.2.3 ロードセグメント飲み取り出す
    CopyLoadSegments(kernel_ehdr);
    Print(L"Kernel: 0x%0lx - 0x%0lx\n", kernel_first_addr, kernel_last_addr);

// 3.2.4 割り当てたメモリを開放する
    status = gBS->FreePool(kernel_buffer);
    if (EFI_ERROR(status)) {
        Print(L"failed to free pool: %r\n", status);
        Halt();
    }

// 4. ファイルシステムをメモリにコピーする

// 4.1 \fat_diskがあればファイトして開く
    VOID *volume_image;                         // => オンメモリFAT fsとして使用する

    EFI_FILE_PROTOCOL *volume_file;
    status = root_dir->Open(
        root_dir, &volume_file, L"\\fat_disk",
        EFI_FILE_MODE_READ, 0);
    if (status == EFI_SUCCESS) {
// 4.1.1 FATボリュームを読み込む
        status = ReadFile(volume_file, &volume_image);
        if (EFI_ERROR(status)) {
            Print(L"failed to read volume file: %r\n", status);
            Halt();
        }
    } else {
// 4.2 \fat_diskがなければdisk.imgをEFI_BLOCK_IO_PROTOCOLとして開く
        EFI_BLOCK_IO_PROTOCOL *block_io;
        status = OpenBlockIoProtocolForLoadedImage(image_handle, &block_io);
        if (EFI_ERROR(status)) {
            Print(L"failed to open Block I/O Protocol: %r\n", status);
            Halt();
        }
// 4.2.1 コピーする容量を求める（上限は32MB）
        EFI_BLOCK_IO_MEDIA *media = block_io->Media;
        UINTN volume_bytes = (UINTN)media->BlockSize *(media->LastBlock + 1);
        if (volume_bytes > 32 * 1024 * 1024) {
            volume_bytes = 32 * 1024 * 1024;
        }

        Print(L"Reading %lu bytes (Present %d, BlockSize %u, LastBlock %u)\n",
            volume_bytes, media->MediaPresent, media->BlockSize, media->LastBlock);

// 4.2.2 ボリュームの一部をコピーする
        status = ReadBlocks(block_io, media->MediaId, volume_bytes, &volume_image);
        if (EFI_ERROR(status)) {
            Print(L"failed to read blocks: %r\n", status);
            Halt();
        }
    }

// 5. BootSefvicesを抜ける: カーネルに移行するために必要
    status = gBS->ExitBootServices(image_handle, memmap.map_key);
    if (EFI_ERROR(status)) {
// 5.1 抜けられなかったらメモリマップを再取得する
        status = GetMemoryMap(&memmap);
        if (EFI_ERROR(status)) {
            Print(L"failed to get memory map: %r\n", status);
            Halt();
        }
// 5.1.1 再度BootSefvicesを抜ける
        status = gBS->ExitBootServices(image_handle, memmap.map_key);
        if (EFI_ERROR(status)) {
            Print(L"Could not exit boot service: %r\n", status);
            Halt();
        }
    }

// 6. カーネルに処理を渡す

// 6.1 カーネルのエントリポイントを取得する
    UINT64 entry_addr = *(UINT64 *)(kernel_first_addr + 24);

// 6.2 フレームバッファ情報を取得し、カーネルに渡す変数に設定する=>画面描画に使用する
    struct FrameBufferConfig config = {
        (UINT8 *)gop->Mode->FrameBufferBase,
        gop->Mode->Info->PixelsPerScanLine,
        gop->Mode->Info->HorizontalResolution,
        gop->Mode->Info->VerticalResolution,
        0
    };

    switch (gop->Mode->Info->PixelFormat) {
        case PixelRedGreenBlueReserved8BitPerColor:
            config.pixel_format = kPixelRGBResv8BitPerColor;
            break;
        case PixelBlueGreenRedReserved8BitPerColor:
            config.pixel_format = kPixelBGRResv8BitPerColor;
            break;
        default:
            Print(L"Unimplemented pixel format: %d\n", gop->Mode->Info->PixelFormat);
            Halt();
    }

// 6.3 ACPI情報を取得し、カーネルに渡す変数に設定する=>タイマーを利用する
    VOID* acpi_table = NULL;
    for (UINTN i = 0; i < system_table->NumberOfTableEntries; ++i) {
        if (CompareGuid(&gEfiAcpiTableGuid,
                        &system_table->ConfigurationTable[i].VendorGuid)) {
            acpi_table = system_table->ConfigurationTable[i].VendorTable;
            break;
        }
    }

// 6.4 カーネルエントリポイントの関数型定義を行う（UEFIのABIはMacとは異なるため属性を付けてMacのABIに合わせる）
    typedef void __attribute__((sysv_abi)) EntryPointType(
        const struct FrameBufferConfig *,
        const struct MemoryMap *,
        const VOID *,
        VOID *);
    EntryPointType *entry_point = (EntryPointType *)entry_addr;
// 6.5 カーネルを呼び出す
    entry_point(&config, &memmap, acpi_table, volume_image);

    Print(L"All done\n");

    while (1);
    return EFI_SUCCESS;
}
```

## 2. 各種処理関数

- 次の2つのグローバル変数が提供されており、自由に使用できる (MdePkg/Include/Library/UefiBootServicesTableLib.h)
  - gBS: EFI_BOOT_SERVICESオブジェクトへのポインタで、EFI_BOOT_SERVICESが提供する各種関数を使用できる
  - gST: EFI_SYSTEM_TABLEオブジェクトへのポインタで、EFI_SYSTEM_TABLEが提供する各種関数を使用できる

### 2.1 `EFI_STATUS GetMemoryMap(struct MemoryMap *map)`

メモリマップを取得する

```cpp
EFI_STATUS GetMemoryMap(struct MemoryMap *map) {
    if (map->buffer == NULL) {
        return EFI_BUFFER_TOO_SMALL;
    }

    map->map_size = map->buffer_size;
    return gBS->GetMemoryMap(               // 7.2 Memory Allocation Services
        &map->map_size,                     // INOUT変数
        (EFI_MEMORY_DESCRIPTOR *)map->buffer,
        &map->map_key,
        &map->descriptor_size,
        &map->descriptor_version);
}
```

#### MemoryMap構造体定義(MikanLoderPkg/memory_map.hpp)

    ```cpp
    struct MemoryMap {
        unsigned long long buffer_size;
        void *buffer;
        unsigned long long map_size;
        unsigned long long map_key;
        unsigned long long descriptor_size;
        uint32_t descriptor_version;
    };
    ```

### 2.2 `EFI_STATUS OpenRootDir(EFI_HANDLE image_handle, EFI_FILE_PROTOCOL **root)`

disk.imgのルートディレクトリを取得する

```cpp
EFI_STATUS OpenRootDir(EFI_HANDLE image_handle, EFI_FILE_PROTOCOL **root) {
    EFI_STATUS status;
    EFI_LOADED_IMAGE_PROTOCOL *loaded_image;
    EFI_SIMPLE_FILE_SYSTEM_PROTOCOL *fs;

    // 1. ロード済みイメージプロトコルを取得
    status = gBS->OpenProtocol(                     // 7.3 Protocol Handler Services
        image_handle,
        &gEfiLoadedImageProtocolGuid,
        (VOID**)&loaded_image,                      // ここで取得したimage protocolを
        image_handle,
        NULL,
        EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);
    if (EFI_ERROR(status)) {
        return status;
    }

    // 2. ファイルシステムプロトコルを取得
    status = gBS->OpenProtocol(
        loaded_image->DeviceHandle,                 // ここで使用
        &gEfiSimpleFileSystemProtocolGuid,
        (VOID**)&fs,                                // ここでfsを取得
        image_handle,
        NULL,
        EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);
    if (EFI_ERROR(status)) {
        return status;
    }

    // 3. fsのルートディレクトリをOpen
    return fs->OpenVolume(fs, root);                // 13.4 Simple File System Protocol
}
```

### 2.3 `EFI_STATUS SaveMemoryMap(struct MemoryMap *map, EFI_FILE_PROTOCOL *file)`

メモリマップをファイルに書き出す

```cpp
EFI_STATUS SaveMemoryMap(struct MemoryMap *map, EFI_FILE_PROTOCOL *file) {
    EFI_STATUS status;
    CHAR8 buf[256];
    UINTN len;

    // 1. ヘッダ行書き込み
    CHAR8 *header =
        "Index, Type, Type(name), PhysicalStart, NumberOfPages, Attribute\n";
    len = AsciiStrLen(header);                                  // MdePkg/Library/BaseLib/String.c
    status = file->Write(file, &len, header);                   // 13.5 File Protocol
    if (EFI_ERROR(status)) {
        return status;
    }

    Print(L"map->buffer = %08lx, map->map_size = %08lx\n",
        map->buffer, map->map_size);
    // 2. メモリマップ行を書き込み
    EFI_PHYSICAL_ADDRESS iter;
    int i;
    for (iter = (EFI_PHYSICAL_ADDRESS)map->buffer, i = 0;
        iter < (EFI_PHYSICAL_ADDRESS)map->buffer + map->map_size;
        iter += map->descriptor_size, i++) {
        EFI_MEMORY_DESCRIPTOR *desc = (EFI_MEMORY_DESCRIPTOR *)iter;
        len = AsciiSPrint(                                      // MdePkg/Library/BasePrintLib/PrintLib.c
            buf, sizeof(buf),
            "%u, %x, %-ls, %08lx, %lx, %lx\n",
            i, desc->Type, GetMemoryTypeUnicode(desc->Type),
            desc->PhysicalStart, desc->NumberOfPages,
            desc->Attribute & 0xffffflu);
        status = file->Write(file, &len, buf);                  // 13.5 File Protocol
        if (EFI_ERROR(status)) {
            return status;
        }
    }

    return EFI_SUCCESS;
}
```

### 2.4 `EFI_STATUS OpenGOP(EFI_HANDLE image_handle, EFI_GRAPHICS_OUTPUT_PROTOCOL **gop)`

Graphic Output Protocolを取得する

```cpp
EFI_STATUS OpenGOP(EFI_HANDLE image_handle,
                EFI_GRAPHICS_OUTPUT_PROTOCOL **gop){
    EFI_STATUS status;
    UINTN num_gop_handles = 0;          // gop_handlesのサイズ
    EFI_HANDLE *gop_handles = NULL;

    // 1. GraphicOutputプロトコルのハンドルを検索して取得する
    status = gBS->LocateHandleBuffer(   // 7.3 Protocol Handler Services
        ByProtocol,
        &gEfiGraphicsOutputProtocolGuid,
        NULL,
        &num_gop_handles,
        &gop_handles);
    if (EFI_ERROR(status)) {
        return status;
    }

    // 2. 1で取得したハンドルでGraphicOutputプロトコルを取得する
    status = gBS->OpenProtocol(         // 7.3 Protocol Handler Services
        gop_handles[0],
        &gEfiGraphicsOutputProtocolGuid,
        (VOID **)gop,
        image_handle,
        NULL,
        EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);
    if (EFI_ERROR(status)) {
        return status;
    }

    FreePool(gop_handles);

    return EFI_SUCCESS;
}
```

### 2.5 `EFI_STATUS ReadFile(EFI_FILE_PROTOCOL *file, VOID **buffer)`

ファイル情報からファイルサイズを取得し、必要なメモリをバッファに割り当て、ファイルを読み込む

```cpp
EFI_STATUS ReadFile(EFI_FILE_PROTOCOL *file, VOID **buffer) {
    EFI_STATUS status;

    UINTN file_info_size = sizeof(EFI_FILE_INFO) + sizeof(CHAR16) * 12; // FineName分を足す
    UINT8 file_info_buffer[file_info_size];
    // 1. ファイル情報の取得
    status = file->GetInfo(                         // 13.5 File Protocol
        file, &gEfiFileInfoGuid,
        &file_info_size, file_info_buffer);
    if (EFI_ERROR(status)) {
        return status;
    }

    EFI_FILE_INFO *file_info = (EFI_FILE_INFO *)file_info_buffer;
    UINTN file_size = file_info->FileSize;

    // 2. 必要なメモリをバッファに割り当てる
    status = gBS->AllocatePool(EfiLoaderData, file_size, buffer);   // 7.2 Memory Allocation Services
    if (EFI_ERROR(status)) {
        return status;
    }
    // 3. ファイルをバッファに読み込む
    return file->Read(file, &file_size, *buffer);   // 13.5 File Protocol
}
```

### 2.6 `void CalcLoadAddressRange(Elf64_Ehdr *ehdr, UINT64 *first, UINT64 *last)`

ロードセグメントアドレスの最小値と最大値を求める

```cpp
void CalcLoadAddressRange(Elf64_Ehdr *ehdr, UINT64 *first, UINT64 *last) {
    Elf64_Phdr *phdr = (Elf64_Phdr *)((UINT64)ehdr + ehdr->e_phoff);
    *first = MAX_UINT64;
    *last  = 0;
    for (Elf64_Half i = 0; i < ehdr->e_phnum; ++i) {
        if (phdr[i].p_type != PT_LOAD) continue;

        *first = MIN(*first, phdr[i].p_vaddr);
        *last  = MAX(*last, phdr[i].p_vaddr + phdr[i].p_memsz);
    }
}
```

### 2.7 `void CopyLoadSegments(Elf64_Ehdr *ehdr)`

ロードセグメントの仮想アドレスにデータをコピーする(bssセグメントは0詰めする)

```cpp
void CopyLoadSegments(Elf64_Ehdr *ehdr) {
    Elf64_Phdr *phdr = (Elf64_Phdr *)((UINT64)ehdr + ehdr->e_phoff);
    for (Elf64_Half i = 0; i < ehdr->e_phnum; ++i) {
        if (phdr[i].p_type != PT_LOAD) continue;
        // copy program header
        UINT64 segm_in_file = (UINT64)ehdr + phdr[i].p_offset;
        CopyMem((VOID *)phdr[i].p_vaddr, (VOID *)segm_in_file, phdr[i].p_filesz);
        // set 0 to bss
        UINTN remain_bytes = phdr[i].p_memsz - phdr[i].p_filesz;
        SetMem((VOID *)(phdr[i].p_vaddr + phdr[i].p_filesz), remain_bytes, 0);
    }
}
```

### 2.8 `EFI_STATUS OpenBlockIoProtocolForLoadedImage(EFI_HANDLE image_handle, EFI_BLOCK_IO_PROTOCOL **block_io)`

disk.img（ロード済みイメージ）からブロックIOを取得する

```cpp
EFI_STATUS OpenBlockIoProtocolForLoadedImage(
    EFI_HANDLE image_handle, EFI_BLOCK_IO_PROTOCOL **block_io) {
    EFI_STATUS status;
    EFI_LOADED_IMAGE_PROTOCOL *loaded_image;

    status = gBS->OpenProtocol(
        image_handle,
        &gEfiLoadedImageProtocolGuid,
        (VOID **)&loaded_image,
        image_handle,
        NULL,
        EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);
    if (EFI_ERROR(status)) {
        return status;
    }

    status = gBS->OpenProtocol(
        loaded_image->DeviceHandle,
        &gEfiBlockIoProtocolGuid,
        (VOID **)block_io,
        image_handle,   // agent handle
        NULL,
        EFI_OPEN_PROTOCOL_BY_HANDLE_PROTOCOL);

    return status;
}
```

### 2.9 `EFI_STATUS ReadBlocks(EFI_BLOCK_IO_PROTOCOL *block_io, UINT32 media_id, UINTN read_bytes, VOID **buffer)`

バッファにメモリを割り当て、ブロックIOからデータをバッファに読み込む

```cpp
EFI_STATUS ReadBlocks(
        EFI_BLOCK_IO_PROTOCOL *block_io, UINT32 media_id,
        UINTN read_bytes, VOID **buffer) {
    EFI_STATUS status;

    status = gBS->AllocatePool(EfiLoaderData, read_bytes, buffer);
    if (EFI_ERROR(status)) {
        return status;
    }

    status = block_io->ReadBlocks(
        block_io,
        media_id,
        0,  // start LBA
        read_bytes,
        *buffer);

    return status;
}
```

## メモリマップ関係の構造体定義

```cpp
tsruct MemoryMap {
    unsigned long long buffer_size;
    void *buffer;
    unsigned long long map_size;
    unsigned long long map_key;
    unsigned long long descriptor_size;
    uint32_t descriptor_version;
};

typedef struct {
    UINT32 Type;
    EFI_PHYSICAL_ADDRESS PhysicalStart;
    EFI_VIRTUAL_ADDRESS VirtualStart;
    UINT64 NumberOfPages;
    UINT64 Attribute;
} EFI_MEMORY_DESCRIPTOR;
```

## ファイル情報構造体

```cpp
typedef struct {
    UINT64 Size;
    UINT64 FileSize;
    UINT64 PhysicalSize;
    EFI_TIME CreateTime;
    EFI_TIME LastAccessTime;
    EFI_TIME ModificationTime;
    UINT64 Attribute;
    CHAR16 FileName[];
} EFI_FILE_INFO;
```

## EFI_GRAPHICS_OUTPUT_PROTOCOL: 12.9 Graphics Output Protocol

ビデオモードの設定や、グラフィックスコントローラのフレームバッファとの間でピクセルをコピーするための基本的な抽象化された機能を提供する。ハードウェアのフレームバッファのリニアアドレスも公開するのでアプリケーションでビデオハードウェアに直接書き込むことができる。

```cpp
typedef struct EFI_GRAPHICS_OUTPUT_PROTCOL {
    EFI_GRAPHICS_OUTPUT_PROTOCOL_QUERY_MODE QueryMode;
    EFI_GRAPHICS_OUTPUT_PROTOCOL_SET_MODE SetMode;
    EFI_GRAPHICS_OUTPUT_PROTOCOL_BLT Blt;
    EFI_GRAPHICS_OUTPUT_PROTOCOL_MODE *Mode;
} EFI_GRAPHICS_OUTPUT_PROTOCOL;

typedef struct {
    UINT32 MaxMode;
    UINT32 Mode;
    EFI_GRAPHICS_OUTPUT_MODE_INFORMATION *Info;
    UINTN SizeOfInfo;
    EFI_PHYSICAL_ADDRESS FrameBufferBase;
    UINTN FrameBufferSize;
} EFI_GRAPHICS_OUTPUT_PROTOCOL_MODE;

typedef struct {
    UINT32 Version;
    UINT32 HorizontalResolution;
    UINT32 VerticalResolution;
    EFI_GRAPHICS_PIXEL_FORMAT PixelFormat;
    EFI_PIXEL_BITMASK PixelInformation;
    UINT32 PixelsPerScanLine;
} EFI_GRAPHICS_OUTPUT_MODE_INFORMATION;
```
