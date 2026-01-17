---
title: "Neovimデビューしたい人必見！｜Neovim公式チュートリアルを日本語でやる方法"
emoji: "🔰"
type: "tech"
topics: ["neovim", "vim", "tutorial", "初心者"]
published: true
---

## はじめに
こんにちは！
｢Neovimの｣公式チュートリアルを日本語で行う方法について解説します！
つい先日NeovimをインストールしたNeovim初心者の私．

>｢Neovimのチュートリアルは英語版しかないから，
>日本語でやりたければVimTutorの日本語版をやるしかない｣

そう思っていたのですが，偶然Neovimにも日本語のチュートリアルが存在することを発見したので共有します！~~(ちなみに内容はvimのチュートリアルとほとんど一緒でした)~~ 

### 対象読者
- これからNeovimを始めたい方
- Neovim操作の日本語教材を探している方
### 記事を読むメリット
- Neovimチュートリアルを日本語でやる方法を知れる
### 結論
Neovimを開き，以下のコマンドを入力してENTERしてください．
```: Neovim内で入力/実行
:Tutor ja/vim-01-beginner.tutor
```
:::message
環境によってはチュートリアルではなく，単なる空のファイルが開かれる場合があります．
:::
その場合はお使いの環境に合わせて以下の手順を試してみてください．

## WSL-Ubuntu (Linux) 環境
### 1. チュートリアルファイルを探す
私の環境では，以下の場所に置かれていました．
```: パス
\wsl.localhost\Ubuntu\usr\share\nvim\runtime\tutor\ja\vim-01-beginner.tutor
```
(同じ場所に，`vim-02-beginner.tutor`という追加チュートリアルも発見しました！)
:::message
OSやインストール方法により，パスは異なる場合があります．
:::
以下のコマンドを実行することによりruntimeフォルダの場所を知ることができます．
```Neovim内で入力/実行
:echo $VIMRUNTIME
```
見つけたruntimeディレクトリの中に，tutorディレクトリがあり，その中にenやjaなどという言語ごとのディレクトリがあると思います．jaの中を調べて，vim-01-beginner.tutorというファイルがあれば，それこそが目的のファイルです！

ない場合は，GitHubから手動で入手し，配置する必要がありそうです．こちらについては実際に検証できていないので，ご参考までに！

https://github.com/neovim/neovim/tree/master/runtime/tutor/ja

### 2. チュートリアルファイルを開く
チュートリアルファイルがあることが確認できたら，いよいよ開いてみます．Neovimを開いて，以下のコマンドを入力し，ENTERしてみてください．
```: Neovim内で入力/ 実行
:Tutor
```
私の環境だと，英語版のチュートリアルが開きました．開いて欲しいのは日本語版のほうです．
そこで直接日本語版のチュートリアルを指定してみましょう．以下のコマンドで指定できます！
```: Neovim内で入力/実行
:Tutor ja/vim-01-beginner.tutor
```
これで日本語版のチュートリアルが開けば成功です！
### 参考 : 繰り返し学習したい方向け
毎回長いコマンドを打つのが大変という場合は，お使いの shellの設定ファイルにエイリアス登録すると便利です！以下はBashにエイリアスを設定する手順の例です．
1. ホームディレクトリにある.bashrcを開きます．(この場合はnanoエディタで開かれます)
```shell : shellで入力/実行
nano ~/.bashrc
```
3. ファイルの末尾に以下の行を追加します．
```shell
# --- Neovim Tutor Aliases ---
alias nvimtutor1='nvim -c "Tutor ja/vim-01-beginner"'
alias nvimtutor2='nvim -c "Tutor ja/vim-02-beginner"'
```
5. 保存して終了し，設定を反映させます．
```shell : shellで入力/実行
source ~/.bashrc
```
6. `nvimtutor1`，`nvimtutor2`と打って，日本語版チュートリアルのチャプター1とチャプター2がそれぞれ開くことを確認してください．これは毎回開くたびに新品の状態になる(原本は編集されない)ので，安心して繰り返し練習してください！

## Windows環境
### 1. チュートリアルファイルを探す
私の環境では，以下の場所に置かれていました．
```: パス
C:\ Program Files\Neovim\share\nvim\runtime\tutor\ja\vim-01-beginner.tutor
```
:::message
OSやインストール方法により，パスは異なる場合があります．
:::
以下のコマンドを実行することによりruntimeフォルダの場所を知ることができます．
```: Neovim内で入力/実行
:echo $VIMRUNTIME
```
見つけたruntimeフォルダの中に，tutorフォルダがあり，その中にenやjaなどという言語ごとのフォルダがあると思います．jaの中を調べて，vim-01-beginner.tutorというファイルがあれば，それこそが目的のファイルです！

ない場合は，GitHubから手動で入手し，配置する必要がありそうです．こちらについては実際に検証できていないので，ご参考までに！

https://github.com/neovim/neovim/tree/master/runtime/tutor/ja

### 2. チュートリアルファイルを開く
チュートリアルファイルがあることが確認できたら，いよいよ開いてみます．Neovimを開いて，以下のコマンドを入力し，ENTERしてみてください．
```: Neovim内で入力/実行
:Tutor
```
私の環境では，チュートリアルファイルではなく，空のファイルが開かれてしまいました．その場合，以下のコマンドを実行してみてください．
```: Neovim内で入力/実行
:edit $VIMRUNTIME/tutor/ja/vim-01-beginner.tutor
```
:::message alert
この方法で開いた場合，原本を直接開くことになる (:Tutorで開いた場合は開き直すと新品の状態に戻るが，この場合はそうならない) ので注意してください．
:::
繰り返し学習する場合は，手動でコピーを作成するか，複製して一時ファイルで開くようなエイリアスを登録することを推奨いたします！
### 参考 : 繰り返し学習したい方向け
毎回長いコマンドを打つのが大変という場合は，お使いのshellの設定ファイルにエイリアス登録すると便利です！以下はPowerShellに関数として登録する手順の例です．
1. 以下のコマンドを実行すると，プロファイルファイルがない場合は作成されます．
```shell: shellで入力/実行
if (!(Test-Path $PROFILE)) { New-Item -ItemType File -Path $PROFILE -Force }
```
2. 以下のコマンドを実行すると，メモ帳で開かれます．
```shell: shellで入力/実行
notepad $PROFILE
```
3. 以下の内容を追記してください．
```shell
function nvimtutor {
    # 1. Fixed path
    $vimruntime = "C:\ Program Files\Neovim\share\nvim\runtime"
    $source = Join-Path $vimruntime "tutor\ja\vim-01-beginner.tutor"

    # 2. Check if source exists
    if (!(Test-Path -LiteralPath $source)) {
        Write-Host "Error: Japanese tutor file not found." -ForegroundColor Red
        Write-Host "Location: $source"
        return
    }

    # 3. Copy to temp (reset every time)
    $temp = "$env:TEMP\vimtutor_practice.tutor"
    Copy-Item -LiteralPath $source -Destination $temp -Force

    # 4. Open copy
    Write-Host "Starting tutorial..." -ForegroundColor Green
    # Use single quotes for the nvim command argument to avoid confusion
    nvim $temp -c 'set nonumber norelativenumber'
}
```
4. `nvimtutor`と打って，日本語チュートリアル(チャプター1)のコピーが開かれることを確認してください．これは一時ファイルとして開かれるので，次回コマンドを打った際には新品の状態に戻ります！

## まとめ
最後まで読んでいただきありがとうございました！
この記事がNeovimデビューする方の参考になれば幸いです．
それでは楽しいNeovimライフを！