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

## [第16章: コマンド](day16.md)

- ターミナルに入力機能を追加
- `echo`, `clear`, `lspci`コマンドを実装
- コマンド履歴機能を実装
- TaskBを削除

## [第17章: ファイルシステム](day17.md)

- UEFIの機能でボリュームデータをメモリに読み込んで使用する
- FAT32の仕様書の解説
- lsコマンドを実装

## [第18章: アプリケーション](day18.md)

- ファイルからアプリケーションを呼び出して実行
  - アセンブリ言語による
  - C言語による（ライブラリ関数を自作）
  - C言語による（標準ライブラリを使用）

## [第19章: ページング](day19.md)

- アドレス変更
  - 仮想アドレスの物理ページへの変換
  - 4階層ページング
- アプリケーション（高位アドレスに配置）のロード・実行
  - アプリケーション実行でシステムがリセットする問題発生
  - lldbによるデバッグ

## [第20章: システムコール](day20.md)

- ユーザ権限のセグメントの作成、ユーザスタックの割当
- TSSによるカーネルスタックの設定
- 例外ハンドラの設定によるデバッグ出力
- MSRを介したシステムコールの実現

## [第21章: アプリからウィンドウ](day21.md)

- newlibを使うためのグルー関数とシステムコール
- アプリケーションからカーネルに戻るExitシステムコール
- アプリケーションからWindowを開き、文字を描くシステムコール

## [第22章: グラフィックとイベント (1)](day22.md)

- 画面に線を描画するアルゴリズム
- 大量描画を最適化する方法
- システムコール時にOS用スタックに入れ替える方法

## [第23章: グラフィックとイベント (2)](day23.md)

- マウス入力、キーボード押下の取得
- タイマ作成システムコール
- eye, paint, timer, アニメーション、ブロック崩しアプリケーションの作成

## [第24章: 複数のターミナル](day24.md)

- アプリ専用の階層ページング機構を用意して複数アプリの同時起動を可能にする
- ターミナルに表示フラグをもたせてターミナルなしにアプリを起動する
- アプリが例外処理を行った場合に終了させる。

## [第25章: アプリでファイル読み込み](day25.md)

- ディレクトリ操作を可能にする
- fopen, freadの実装（open, readシステムコールの実装）

## [第26章: アプリでファイル書き込み](day26.md)

- ファイルディスクリプタを抽象化し、stdin, stdout, stderrを実装
- fwriteの実装（writeシステムコールの実装）

## [第27章: アプリのメモリ管理](day27.md)

- デマンドページングの実装
- メモリマップトファイルの実装
- コピーオンライトの実装

## [Macでの環境構築](mac.md)
