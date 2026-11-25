---
title: "Echo Show 5を脱獄し自作AI秘書VoxMateを作る全手順"
emoji: "🎙️"
type: "tech"
topics: ["ai", "voicevox", "whisper", "android", "python"]
published: false
---

机の隅で眠っていた Echo Show 5（第1世代）を、ローカル音声認識（Whisper）＋ LLM ＋ VOICEVOX ＋ 立ち絵キャラクターが動く **専用の AI 音声アシスタント端末** に作り変えました。

この記事を読むと、**Echo Show 5 をカスタム ROM 化して Web 端末として再利用する手順** と、**0.2 秒起動のウェイクワード・VOICEVOX 音素リップシンクを備えた音声アシスタント「VoxMate」を自分の環境で動かす手順** が分かります。

![VoxMate の実機動作デモ（立ち絵・リップシンク・表情連動）](/images/echo-show-5-voxmate-assistant/demo.gif)

公開リポジトリは以下です。常時運用しやすい公式 Gemini API バックエンド（`v0.1.3`）も同梱しています。

https://github.com/Naruto-Takahashi/voxmate

## 全体構成

Echo Show 5（第1世代）はメモリが小さいため、端末側は Chrome で画面表示とマイク・スピーカー入出力だけを担当させ、音声認識・LLM・VOICEVOX はすべて母艦の Mac mini（M1 8GB）で処理します。

![VoxMate + Echo Show 5 全体アーキテクチャ](/images/echo-show-5-voxmate-assistant/architecture.png)

## 手順1：Echo Show 5 第1世代のアンロックと LineageOS 18.1 導入

Echo Show 5 第1世代（コードネーム `checkers`）は、一度塞がれたブートローダーアンロック用脆弱性（`fastbrick`）が後のビルド `Fire OS 6575.1 (NS65751/8851)` でリバートされ、再アンロックできることが XDA Developers で報告されています。

:::message alert
**改造・アンロックに関する注意**
ブートローダーアンロックおよびカスタム ROM 導入はメーカー保証の対象外となり、手順を誤ると端末が起動しなくなる（文鎮化する）リスクがあります。実施する場合は自己責任のもと、必ず XDA の一次スレッドで最新手順と対象ビルドを確認してください。なお、ツールの再配布は行いません。
:::

1. **対象ビルドの確認と OTA 遮断**: 「設定」→「デバイスオプション」→「デバイス情報」で `Fire OS 6575.1 (NS65751/8851)` であることを確認したら、**即座に Wi-Fi をオフ** にして自動アップデートを止めます。
2. **Windows でアンロックと TWRP 導入**: XDA スレッドから `amonet-checkers-v2.0.2` と LineageOS 18.1 イメージを入手し、Google USB Driver を入れた Windows PC で `fastbrick.bat` を実行します。指示に従って USB 接続し、アンロック後に TWRP を起動します。
3. **LineageOS 18.1 の焼成**: TWRP の「Wipe」→「Advanced Wipe」で Data / Cache を初期化したうえで、ROM を転送してインストールします。

```bash:lineageos-flash.sh
adb push lineage-18.1-*-UNOFFICIAL-checkers.zip /sdcard/
# TWRP の「Install」から上記 ZIP を選択して焼成し、再起動する
```

4. **USB デバッグ有効化**: 起動後、「設定」→「端末情報」→「ビルド番号」を 7 回タップして開発者向けオプションを開き、USB デバッグを ON にします。最後に Chrome を入れて端末側の準備は完了です。

https://xdaforums.com/t/unlock-root-twrp-unbrick-echo-show-5-checkers.4718281/

## 手順2：母艦（Mac mini / PC）に VoxMate をセットアップする

母艦側では Python 3.12+、[`uv`](https://docs.astral.sh/uv/)、`ffmpeg`、`jq`、`adb` を用意します。

### 2-1. クローンと依存パッケージの同期

```bash:setup-voxmate.sh
git clone https://github.com/Naruto-Takahashi/voxmate.git
cd voxmate
uv sync
```

Apple Silicon Mac では `mlx-whisper`、Linux / Windows では `faster-whisper` が自動で選択されます。

### 2-2. VOICEVOX Engine の準備

- **macOS (Apple Silicon)**: [VOICEVOX Engine Releases](https://github.com/VOICEVOX/voicevox_engine/releases) から `voicevox_engine-macos-arm64-*.tar.gz` を取得し、解凍した中身を `voicevox/macos-arm64/` に配置します（`./start.sh` が自動起動します）。
- **Linux / Windows**: Docker 等で `http://127.0.0.1:50021` に待ち受けさせます。

```bash:voicevox-docker.sh
docker run --rm -p 50021:50021 voicevox/voicevox_engine:cpu-latest
```

### 2-3. AI バックエンド設定（`.env`）

`VOXMATE_BACKEND` で `gemini` / `claude` / `agy` を切り替えられます。

:::message alert
**AI バックエンド選定と利用規約上の注意**
開発中は手元の Antigravity CLI（`agy`）や Claude Code CLI（`claude`）でも検証しましたが、Google Antigravity の利用規約（`antigravity.google/terms`）にはサードパーティ製ツールとの接続を制限する条項があります。常時稼働の音声アシスタントとして安全に運用する場合は、公式 API キーを使う **Gemini API バックエンド（`VOXMATE_BACKEND=gemini`）** を推奨します。
:::

`.env.example` をコピーして `.env` を作成し、API キーを設定します。

```bash:.env
VOXMATE_BACKEND=gemini
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-2.5-flash

VOICEVOX_URL=http://127.0.0.1:50021
VOICEVOX_SPEAKER=3
```

### 2-4. 起動と端末からの接続

```bash:run-voxmate.sh
# macOS（VOICEVOX Engine + VoxMate + adb reverse を一括維持）
./start.sh

# Linux / Windows（外部で VOICEVOX Engine 起動済みの場合）
uv run python -m uvicorn server:app --host 127.0.0.1 --port 8000
```

- **Echo Show 5**: USB 接続した状態で Chrome から `http://localhost:8000` を開きます（`adb reverse` 経由なので証明書なしでマイクが使えます）。
- **スマホ・他 PC**: Tailscale を使っている場合は `tailscale serve --bg 8000` で HTTPS 公開すると、別室や外出先からも同期して使えます。

## 手順3：ウェイクワード学習と立ち絵の抽出（任意）

### 3-1. カスタムウェイクワード（`openWakeWord`）

「ちょっと聞いて」を約 0.2 秒で検知する軽量 ONNX モデル（`models/chotto_kiite.onnx`）を同梱しています。自分の声に合わせて再学習したい場合は、VOICEVOX 合成音声の生成 → `/kws` での生声録音 → 学習の順で実行します。

```bash:train-kws.sh
uv run python tools/kws_generate.py --pos 800 --neg 1200 --out-dir data/kws
# ブラウザで http://localhost:8000/kws を開き、自分の声を録音する
uv run python tools/kws_train.py --data-dir data/kws --out models/chotto_kiite.onnx --epochs 35
```

### 3-2. 立ち絵の抽出（`extract_portraits.py`）

立ち絵なしでもプレースホルダー表示で動きますが、坂本アヒル氏の立ち絵 PSD（各自で規約に従って入手したもの）を `custom_psd/` に置くと、表情・口パク用の差分 PNG を一括抽出できます。

```bash:extract-portraits.sh
uv run python tools/extract_portraits.py --psd-dir custom_psd --out-dir static/portraits
```

あとは「今何時？」「3分後にタイマーかけて」のような定型操作は `fast_path.py` が LLM なしで即答し、自由な会話は LLM の `[expr:...]` 感情タグと VOICEVOX の音素リップシンクで自然に返答してくれます。

:::message
**立ち絵素材・音声合成クレジットとライセンス**
- **立ち絵素材**: 坂本アヒル氏 配布素材（[利用規約・ガイドライン](https://zan-shin.net/guideline/) / [東北ずん子・ずんだもん公式ガイドライン](https://zunko.jp/guideline.html)）。本リポジトリおよび記事内では PSD・PNG 素材の再配布は行いません。
- **音声合成**: [VOICEVOX](https://voicevox.hiroshiba.jp/)
:::

## 参考リンク

https://github.com/Naruto-Takahashi/voxmate

https://xdaforums.com/t/unlock-root-twrp-unbrick-echo-show-5-checkers.4718281/

https://antigravity.google/terms
