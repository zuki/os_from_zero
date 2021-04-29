# 24.1 ターミナルを増やす

- F2が押されたら新しいターミナルを作成
-
![f2](images/day24_f2_terminal.png)

# 24.2 カーソル点滅を自分で

- ターミナルタスクがタイマを制御する
- アクティブウィンドウが切り替わったことをタスクに通知する
- ターミナルが非アクティブになったらカーソル点滅を停止、アクティブになったら再開する

![cursor](images/day24_cursor_control.png)

# 24.3 複数アプリの同時起動

## 現在は複数のアプリが階層ページングを共有しているため競合が発生する

![conflict](images/day24_conflict_paging.png)

## アプリごとに専用の階層ページング機構を持つようにする

- アプリごとに実行可能ファイルをロードする前にPML4を作成・設定する
- 新規PML4を作成する際に、前半部分は現在のcr3からコピーする
  - 前半部分はカーネル用で共通のため
- アプリ終了時にPML4を削除する

![paging per app](images/day24_perapp_paging.png)

# 24.4 ウィンドウの重なりのバグ修正

- 新規作成レイヤのレイヤの設定にバグ

## バグ修正前

![before_fix](images/day24_before_layer_bug.png)

## バグ修正後

![after_fix](images/day24_after_layer_bug.png)

# 24.5 ターミナルなしのアプリ起動

- ターミナルクラスに画面表示フラグを新設する
- 表示フラグが立っている場合のみ画面表示する
- 画面表示しないコマンド(`noterm`)を作成する

![noterm](images/day24_noterm.png)

# 24.6 OSをフリーズさせるアプリ

- アプリが例外を発生させるとOSがフリーズする

![halt](images/day24_halt.png)
![zero](images/day24_zero.png)
![kernel memory](images/day24_kernel_mem_access.png)
![user memory](images/day24_nonpaging_app_access.png)

# 24.8 OSを守ろう

- アプリが例外を発生させた場合はアプリを終了させる
  - 実行中のコードがアプリかカーネル化はCSのCPLをチェックする
  - CPL==3はアプリ、CPL==0はカーネル
- ELF以外の実行ファイルをエラーとする

![kill app](images/day24_kill_app.png)
