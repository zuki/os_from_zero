# 「ゼロからのOS自作入門」自習メモ

## [Linuxでの開発環境の構築](day00.md)

## [第1章: PCの仕組みとハローワールド](day01.md)

- バイナリ入力によるhello world
- UEFIアプリによるhello world

## [第2章: EDK II入門とメモリマップ](day02.md)

- EDK IIを使ったUEFIアプリによるhello world
- UEFI機能を使ったメモリマップの取得

## [第3章: 画面表示の練習とブートローダ](day03.md)

- カーネル(Cプログラム）作成
- ブートローダ（UEFIアプリ）作成
  - ブートローダからカーネルを呼び出して実行
  - ブートローダからカーネルへ引数を渡す
- フレームバッファ（ブートローダで用意してカーネルに渡す）を使ったピクセル描画

## [第4章: ピクセル描画とmake入門](day04.md)

- 配置newによるメモリ確保
- ブートローダの改良（LOADセグメントの選択ロード）

## [第5章: 文字表示とコンソールクラス](day05.md)

- フォントの描画原理
- printk()

## [第6章: マウス入力とPCI](day06.md)

- USBホストドライバ(xHC)、マウスデバイスドライバ(HIDマウス）の導入（説明なし）
- PCIデバイスの解説（PCIデバイスリストの作成）
- xHCの使用法

## [第7章: 割り込みとFIFO](day07.md)

- X86_64割り込みの解説
- FIFO(queue)の実装（イベントキューとして使用）

## [第8章: メモリ管理](day08.md)

- カーネルによるメモリ管理（スタック、セグメンテーション設定、ページング）

## [第9章: 重ね合わせ処理](day09.md)

- sbrk()の実装によるNew演算子の実現
- レイヤー導入
- シャドウバッファによる高速化
- memcpy()による高速化

## [第10章: ウィンドウ](day10.md)

- ウィンドウの作成
- intersection領域の更新による高速化
- バックバッファによる画面のチラツキ防止
- ウィンドウのドラッグ

## [第11章/第12章2節: タイマとACPI](day11.md)

- タイマ割り込みを実現

## [第12章: キー入力](day12.md)

- USBキーボードドライバの作成
- キー押下イベントの処理

## [第13章: マルチタスク (1)](day13.md)

- コンテキストスイッチ
- プリエンプティブマルチタスク

## [第14章: マルチタスク (2)](day14.md)

- SleepとWakeup
- タスク優先度を導入

## [第15章: ターミナル](day15.md)

- ウィンドウ描画を専用タスクで行う
- アクティブウィンドウ機能
- ターミナルを作成（入力機能はまだない）

## [Macでの環境構築](mac.md)
