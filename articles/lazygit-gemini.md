---
title: "LazyGit × Geminiでターミナル環境でもコミットメッセージを自動生成！"
emoji: "📑"
type: "tech" # tech: 技術記事 / idea: アイデア
topics: ["nvim", "git", "gemini", "cli"]
published: true 
---
## はじめに
Gitのコミットメッセージ，何を書こうか迷った経験はありませんか？VSCodeではGitのコミットメッセージをAI生成する機能があるのですが，同じことをターミナル環境でもできたら便利ですよね！そこで本記事では以下の構成で，最強のコミット環境を作る方法を紹介します！
* LazyGit : マウスを使わず，キーボードだけでGitの全操作を完結させるTUI (Text User Interface) クライアント．
* Gemini API : Googleの生成AI (Flashモデル)
* curl : Web APIを叩くためのコマンド．
* jq : APIから返ってきた複雑なデータ (JSON) から，必要なテキストだけを抽出したり，逆にAPIに送るデータを綺麗に整形したりするコマンド．
* gum : ｢ローディングアニメーション｣，｢選択肢リスト｣，｢テキスト入力欄｣などのリッチなUIを与えるツール．

### 対象読者
* ターミナルから出たくない方
* 脱マウスをしたい方
* とはいえもっと気軽にGitを扱いたい方

## 完成した挙動 (使い方)
NeovimからLazyGitを起動し，コミットを行いました．
![](/images/lazygit-gemini/usage.gif)
*実際に使用している様子*
以下でコミットまでの流れを解説します．
1. ステージング
LazyGitを起動し，`j`,`k`や矢印キーで目的のファイルを選択，`SPACE`でステージングします．ここまでは標準の操作です．
![](/images/lazygit-gemini/usage-01.png)
2. コミットメッセージの自動生成
ステージングされた状態で`CTRL + G`を押すとコミットメッセージの自動生成が開始します．英語と日本語それぞれで，簡易版と詳細版の2つずつのコミットメッセージが生成されます．
 ![](/images/lazygit-gemini/usage-02.png)
3. メッセージの選択
生成されたメッセージを`j`，`k`や矢印キーで選択し，`ENTER`で決定します．(中止したい場合は，`CTRL + C`を押したあと`ENTER`を押します．)
![](/images/lazygit-gemini/usage-03.png)
4. コミット
下のようなメッセージの編集画面に移るので，(必要に応じて) 選択したメッセージを編集します．`CTRL + D`で決定され，そのままコミットされます．(中止したい場合は，`CTRL + C`を押します．)
![](/images/lazygit-gemini/usage-04.png)
さらに`ENTER`を押すとLazyGitの画面に戻ります．
![](/images/lazygit-gemini/usage-05.png)

## 仕組み
1. LazyGitからカスタムコマンドとしてシェルスクリプトを呼び出す．
2. スクリプトが`git diff --cached`で変更内容を取得．
3. jqで変更内容をJSONペイロードに整形．
4. curlでGemini API (3.0-Flashモデル) を直接叩く．
5. レスポンス (英語･日本語の候補) をjqでパース．
6. gum chooseで候補を選択させる．
7. gum writeで最終確認･編集画面を表示し，Gitコミットを実行．

## 前提条件
:::message
本記事のスクリプトは**macOS**または**Linux (Windowsの場合はWSL2)** での動作を想定しています．
:::
### 1. LazyGit
#### macOS (Homebrew) の場合
```bash
brew install lazygit
```
#### Linux の場合
```bash
LAZYGIT_VERSION=$(curl -s "https://api.github.com/repos/jesseduffield/lazygit/releases/latest" | grep -Po '"tag_name": "v\K[^"]*')
curl -Lo lazygit.tar.gz "https://github.com/jesseduffield/lazygit/releases/latest/download/lazygit_${LAZYGIT_VERSION}_Linux_x86_64.tar.gz"
tar xf lazygit.tar.gz lazygit
sudo install lazygit /usr/local/bin
```
### 2. その他のツール (jq, gum, curl)
#### macOS (Homebrew) の場合
Homebrewを使えばまとめてインストールできます．
```bash
brew install curl jq gum
```
#### Linux の場合
curlとjqは標準パッケージからインストールします．gumは標準リポジトリに含まれていないことが多いため，開発元 (Charm) から公式リポジトリを追加してインストールします．
```bash
# curl, jq のインストール
sudo apt update && sudo apt install -y curl jq

# gum のインストール (公式リポジトリを利用)
sudo mkdir -p /etc/apt/keyrings
curl -fsSL https://repo.charm.sh/apt/gpg.key | sudo gpg --dearmor -o /etc/apt/keyrings/charm.gpg
echo "deb [signed-by=/etc/apt/keyrings/charm.gpg] https://repo.charm.sh/apt/ * *" | sudo tee /etc/apt/sources.list.d/charm.list
sudo apt update && sudo apt install gum
```

### 3. Google Gemini API Keyの取得
[Google AI Studio](https://aistudio.google.com/app/api-keys) からAPIキーを取得し，環境変数に設定しておきます．
:::message
`<your_api_key>`はコピーしたAPIキーに書き換えてください．
:::
```bash
# .zshrc や .bashrc に追記
export GEMINI_API_KEY="<your_api_key>"
```
これで準備は整いました．それでは実装に移りましょう！

## 実装手順
### 1. 実行スクリプトの作成
`~/.local/bin/lazygit-gemini-commit`というファイルを作成し，以下の内容を記述します．これが今回の心臓部です．
```bash :lazygit-gemini-commit
#!/bin/bash

# ==========================================
# 共通設定 & チェック
# ==========================================

# APIキーの確認 (非対話シェル対策)
if [ -z "$GEMINI_API_KEY" ]; then
  [ -f ~/.zshrc ] && source ~/.zshrc
  [ -f ~/.bashrc ] && source ~/.bashrc
  [ -f ~/.profile ] && source ~/.profile
fi

if [ -z "$GEMINI_API_KEY" ]; then
    gum style --foreground 196 "Error: GEMINI_API_KEY not found."
    read -n 1 -s -r -p "Press any key to exit..."
    exit 1
fi

# diffを取得
DIFF=$(git diff --cached)
if [ -z "$DIFF" ]; then
    gum style --foreground 196 "No staged changes."
    read -n 1 -s -r -p "Press any key to exit..."
    exit 1
fi

# ==========================================
# AIによる候補生成 (英語 & 日本語)
# ==========================================

# プロンプト: 英語と日本語で，簡潔なものと詳細なものの計4パターンを要求
PROMPT="You are a commit message generator. Analyze the following git diff and generate 4 distinct commit messages conforming to Conventional Commits specification.

Generate the following 4 variations:
1. English: A standard, concise message.
2. English: A slightly more detailed message explaining the 'why'.
3. Japanese: A standard, concise message (日本語).
4. Japanese: A slightly more detailed message (日本語).

Output MUST be a valid JSON array of strings, like:
[\"feat: english message\", \"fix: english detailed...\", \"feat: 日本語のメッセージ\", \"fix: 日本語の詳細...\"]

Do NOT output markdown code blocks. Output ONLY the JSON array.

Diff:
$DIFF"

# JSONペイロード作成 (jqを使って安全にエスケープ)
PAYLOAD=$(jq -n --arg prompt "$PROMPT" '{contents: [{parts: [{text: $prompt}]}]}')

# curl引数エラー回避のため一時ファイルを使用
TMP_PAYLOAD=$(mktemp)
echo "$PAYLOAD" > "$TMP_PAYLOAD"
TMP_RESPONSE=$(mktemp)
TMP_ERROR=$(mktemp)

# APIリクエスト (gemini-3-flash-preview)
# gum spin で待機中にスピナーを表示
gum spin --spinner dot --title "Generating candidates..." -- \
    bash -c "curl -s -S -X POST \
      -H 'Content-Type: application/json' \
      -H \"x-goog-api-key: ${GEMINI_API_KEY}\" \
      -d @$TMP_PAYLOAD \
      'https://generativelanguage.googleapis.com/v1beta/models/gemini-3-flash-preview:generateContent' \
      > $TMP_RESPONSE 2> $TMP_ERROR"

RESPONSE=$(cat "$TMP_RESPONSE")
rm "$TMP_PAYLOAD" "$TMP_RESPONSE" "$TMP_ERROR"

# レスポンス解析
CANDIDATES=$(echo "$RESPONSE" | jq -r '.candidates[0].content.parts[0].text // empty')

# Markdownコードブロック除去 (念のため)
CANDIDATES_CLEAN=$(echo "$CANDIDATES" | sed 's/^```json//g' | sed 's/^```//g')

# 配列パースしてgum用の改行区切りテキストに変換
CHOICES=$(echo "$CANDIDATES_CLEAN" | jq -r '.[]' 2>/dev/null)

if [ -z "$CHOICES" ]; then
    gum style --foreground 196 "Failed to parse candidates."
    echo "Raw Response: $CANDIDATES"
    read -n 1 -s -r -p "Press any key to exit..."
    exit 1
fi

# ==========================================
# 候補選択 (gum choose)
# ==========================================
SELECTED_MSG=$(echo "$CHOICES" | gum choose --header "Pick a commit message (English/Japanese)")

if [ -z "$SELECTED_MSG" ]; then
    echo "Cancelled."
    exit 0
fi

# ==========================================
# 最終編集 (gum write)
# ==========================================
# 選択した候補を初期値としてエディタを開く
FINAL_MSG=$(echo "$SELECTED_MSG" | gum write --width 80 --height 5 --placeholder "Edit commit message...")

# 入力があればコミット
if [ -n "$FINAL_MSG" ]; then
    git commit -m "$FINAL_MSG"
else
    echo "Commit cancelled."
fi
```

::::message alert
作成したら，実行権限を付与します．
```bash
chmod +x ~/.local/bin/lazygit-gemini-commit
```
::::

### 2. LazyGitの設定
`~/.config/lazygit/config.yml`にカスタムコマンドを追加します．`subprocess: true`にすることで，LazyGitから端末の制御を一時的にスクリプトへ渡します (これでgumが動きます)．
```yaml :config.yml
customCommands:
  - key: '<c-g>'
    description: 'Generate commit message via Gemini'
    context: 'global'
    subprocess: true
    command: '/home/<your_username>/.local/bin/lazygit-gemini-commit'
```

:::message
`command`のパスはご自身の環境に合わせて書き換えてください．
:::

## おわりに
快適なGitライフを！
