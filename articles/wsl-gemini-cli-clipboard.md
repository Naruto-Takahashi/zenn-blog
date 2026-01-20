--- 
title: "WSL2のGemini CLIにクリップボードの画像を渡す方法"
emoji: "👏"
type: "idea" # tech: 技術記事 / idea: アイデア
topics: ["wsl","gemini","cli"]
published: true 
---
## はじめに
最近，ターミナルで使えるAI，Gemini CLIにハマっているのですが，使っていて少し不便に感じることがありました．それは，クリップボードの画像を渡す機能がついていないことです．
画面で表示されているエラーの内容を伝えたいときなど，スクショを撮って渡したい場面は多くあります．一応スクショ画像のパスを指定して渡すことは可能ですが，どうにも面倒です．
なにか良い方法はないかと調べたところ，なんと，
> Windows版では`ALT+v`でクリップボードの画像を貼れる

という情報を見つけました！これはどうやら最近追加された機能とのことです．これでWindows版では解決ですね．
しかしWSL環境でGemini CLIを使いたい場合，この方法は使えません．WindowsとWSL間で，｢テキスト｣のクリップボード共有は行われる一方，｢画像データ｣のクリップボード共有は行われないからです．
そこで今回，WSL環境でクリップボードの画像を渡す方法を共有いたします！

### 対象読者
- 普段WSL環境で開発を行うことが多いGemini CLIユーザー
- ターミナルから出たくない方
- マウス操作やウィンドウ切り替えを極力減らしたい効率化好きの方

### 記事を読むメリット
- クリップボードの画像を爆速でGemini CLI に渡せるようになる
- WSLとWindows連携の技術的知見が得られる

## 完成した挙動(使い方)
以下のようなワークフローになります．
1. Windows側: スクショを撮る．
2. Gemini CLI: `/c`を実行．
    → 画像がWSL内に取り込まれます．
3. Gemini CLI: `/v`を実行．
    → 画像がプロンプトに添付され，AIと会話できます．

実際のターミナル画面はこんな感じです．
![](/images/wsl-gemini-cli-clipboard/usage-01.png)
*使用例①*
:::message
今回は，`/c`をした時点で **ReadFile** しており，画像を受け取っているので，実は`/v`の必要はありません．しかしこれはGeminiが自律的に｢気を利かせて｣やったことで，毎回そうなるとは限りません．詳しくは，補足にて後述します．
:::

## 仕組み
WSLからWindowsのクリップボードにアクセスするために，`Powershell.exe`を経由します．今回は以下の2ステップ方式を採用しました．
1. Capture (`/c`): Powershell経由でクリップボードの画像をTempに保存し，WSL配下の`.gemini/tmp/`に移動する．
2. View (`/v`): 保存された画像を`@{...}`構文で読み込み，プロンプトに添付する．

なぜわざわざ2つのコマンドに分けたのか？その理由は後述します．

## 実装手順
合計3つのファイルを作成･設定します．
### 手順1. 画像取得スクリプトを作成
#### ファイル①   get-clip-img
PowerShellと連携して画像を保存するBashスクリプトです．
- パス : `~/.local/bin/get-clip-img`

```bash :get-clip-img
#!/bin/bash

# デフォルトの保存先 ($HOMEを使ってユーザー名を自動解決)
OUTPUT_FILE="${1:-$HOME/.cache/clipboard_image.png}"
OUTPUT_DIR=$(dirname "$OUTPUT_FILE")
mkdir -p "$OUTPUT_DIR"

# PowerShellコマンド: クリップボード画像をWindowsのTempに保存
PS_COMMAND=' 
try { 
    Add-Type -AssemblyName System.Windows.Forms
    if ([System.Windows.Forms.Clipboard]::ContainsImage()) {
        $image = [System.Windows.Forms.Clipboard]::GetImage()
        $tempPath = [System.IO.Path]::GetTempFileName()
        $imagePath = $tempPath + ".png"
        $image.Save($imagePath, [System.Drawing.Imaging.ImageFormat]::Png)
        Remove-Item $tempPath
        Write-Output $imagePath
        exit 0
    } else {
        exit 1
    }
} catch {
    exit 1
}
'

# PowerShellを実行してパスを取得
WIN_TEMP_PATH=$(powershell.exe -NoProfile -Command "$PS_COMMAND" | tr -d '\r')

# エラーチェック
if [ $? -ne 0 ] || [ -z "$WIN_TEMP_PATH" ]; then
    echo "Error: No image found."
    exit 1
fi

# WSLパスに変換して移動
WSL_TEMP_PATH=$(wslpath -u "$WIN_TEMP_PATH")
if [ -f "$WSL_TEMP_PATH" ]; then
    mv "$WSL_TEMP_PATH" "$OUTPUT_FILE"
    echo "Image saved to: $OUTPUT_FILE"
else
    echo "Error: File not found."
    exit 1
fi
```
::::message alert
作成したら，以下のコマンドで実行権限を付与しておきます．
```bash
chmod +x ~/.local/bin/get-clip-img
```
::::
----------

### 手順2. GeminiCLIカスタムコマンドを設定
#### ファイル②  c.toml
画像を保存するためのコマンドです．
- パス : `~/.gemini/commands/c.toml`

:::message
以下の `<your-username>` の部分は，あなたのWSLユーザー名に書き換えてください．
※ TOMLファイル内では環境変数が展開されないため，絶対パスでの記述を推奨します．
:::
```toml :c.toml
description = "Capture clipboard image (Step 1)"
prompt = """
!{/home/<your-username>/.local/bin/get-clip-img .gemini/tmp/clipboard_image.png > /dev/null && echo \"Image captured.\" || echo \"Failed to capture image.\"}
"""
```

#### ファイル③  v.toml
画像を読み込んで送信するためのコマンドです．
- パス : `~/.gemini/commands/v.toml`
```toml :v.toml
description = "View captured image (Step 2)"
prompt = """
@{.gemini/tmp/clipboard_image.png}

{{args}}
"""
```

## 技術的な壁と今回とった解決策
実は，最初は｢1つのコマンドで保存から送信までやりたい｣と考えていました．しかし，いくつかの技術的な壁にぶつかり，現在の形に落ち着きました．
#### 壁その1 : セキュリティ制限
最初は画像を`/tmp/`や`~/.cache/`に保存しようとしたのですが，Gemini CLIのセキュリティ制限により，｢ワークスペース(現在のディレクトリ)外のファイルは読み込めない｣というエラーが発生しました．
##### 解決策
保存先をカレントディレクトリ配下の`.gemini/tmp/`という隠しフォルダに変更しました．これにより，Gemini CLIがプロジェクト内のファイルだと認識し，アクセスが可能となります．

----------

#### 壁その2 : 競合状態
最初は1つのコマンド内で`!{画像保存}`と`@{画像読み込み}`を並べて書いていました．しかし，これだと｢画像保存が終わる前に，読み込み処理が走ってしまう｣とう競合状態が生じました．Gemini CLIはおそらくプロンプトをパースする段階で`@{...}`のファイルをチェックするため，まだ保存されていないファイルを読みに行き，エラーになってしまうのだと考えられます．
##### 解決策
非同期処理の制御とパース順序の問題を回避するため，思い切って｢保存(`\c`)｣と｢読み込み(`\v`)｣を分ける設計にしました．人間が`\c`を打ってから`\v`を打つまでの数秒が，確実な待機時間として機能します．

## 補足 : Gemini CLIエージェントの自律動作について

`/c` コマンドを実行した後，明示的に `/v` コマンドを実行しなくても，Gemini が自律的に画像を認識して会話を始められる場合があります．

*   **現象**: `/c` を実行した直後，Gemini が勝手に `.gemini/tmp/clipboard_image.png` を `read_file` ツールで読み込み，画像の内容について言及する．
*   **理由**: Gemini CLI はエージェントとして動作しており，`/c` コマンドの出力（"Image captured." やファイルパス）をコンテキストとして受け取ります．これにより，モデルが「ユーザーは新しく保存されたこの画像について話したいのだろう」と推測し，気を利かせて自発的に `read_file` ツールを呼び出すことがあるためです．
*   **運用**: この挙動はモデルの推論に依存するため，確実ではありません．
    *   Gemini が画像を読み込んだ場合（ログに `✓ ReadFile ...` と出る場合）は，そのまま会話を続けて構いません．
    *   読み込まなかった場合は，手順通り `/v` コマンドを実行してください．

### 自律動作の強化設定 (`GEMINI.md`)
この「気を利かせる」動作の確率を高めるため，システムプロンプトファイル（`.gemini/GEMINI.md`）に以下のルールを追記することを推奨します．

```markdown
---
## クリップボード画像連携 (/c コマンド)
- ユーザーが `/c` コマンドを実行し，出力に "Image captured." が含まれていた場合，あなたは**直ちに** `read_file` ツールを使用して，コマンド内で指定された画像パス（通常は `.gemini/tmp/clipboard_image.png`）を読み込んでください．
- ユーザーからの `/v` コマンド（画像読み込み指示）を待つ必要はありません．能動的に画像を読み込み，その内容を把握してください．
- 読み込み後は，「画像を確認しました」等の短い応答を返し，ユーザーの質問を待ってください．
```

以上の設定を行うことで，`/c`を実行した際に自律的に画像ファイルを読む確率を大幅に上げることができます．これにより，以下のような実質`\c`の1つのコマンドだけでの運用が可能となります．
![](/images/wsl-gemini-cli-clipboard/usage-02.png)
*使用例②*

## おわりに
これで，WSL上のGemini CLIに対しても，GUIのエラー画面などのスクショを手軽に渡せるようになりました．WSL環境でGemini CLIを使っているターミナルユーザーの方は，ぜひ試してみてください！
