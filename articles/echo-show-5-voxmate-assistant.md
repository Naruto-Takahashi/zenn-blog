---
title: "Echo Show 5を脱獄し自作AI秘書VoxMateを作る全手順"
emoji: "🎙️"
type: "tech"
topics: ["ai", "voicevox", "whisper", "android", "python"]
published: false
---

机の隅で眠っていた Echo Show 5（第1世代）を、ローカル音声認識（Whisper）＋ LLM ＋ VOICEVOX ＋ 立ち絵キャラクターが動く **専用の AI 音声アシスタント端末** に作り変えました。

この記事を読むと、**古いスマートディスプレイを文鎮化させずに自作 Web 端末として再利用する手順** と、**0.2 秒起動のカスタムウェイクワード・0 秒台ファストパス・音素リップシンクを備えた音声アシスタント「VoxMate」を自分の環境で立ち上げる全手順** が分かります。

![VoxMate の実機動作デモ（立ち絵・リップシンク・表情連動）](/images/echo-show-5-voxmate-assistant/demo.gif)

:::message
**この記事で作るもの（VoxMate）**
- **端末**: Echo Show 5 第1世代（LineageOS 18.1 + Chrome）＋ PC・スマホで同期
- **音声認識**: Apple Silicon 上の `mlx-whisper` + Silero VAD + 自前学習した `openWakeWord`（「ちょっと聞いて」）
- **頭脳**: 0 秒台で即答する決定論的ファストパス ＋ 差し替え可能な LLM バックエンド（公式 Gemini API / Claude Code / Antigravity CLI）
- **表現**: VOICEVOX 音声合成 ＋ 音素タイムライン連動の5母音リップシンク ＋ 感情タグ連動の立ち絵モーション
:::

公開リポジトリは以下です。利用規約面でクリーンに常時運用しやすい公式 Gemini API バックエンド（`v0.1.3`）も同梱しています。

https://github.com/Naruto-Takahashi/voxmate

## 全体アーキテクチャと端末ごとの役割分担

Echo Show 5（第1世代）はメモリ 1GB 前後の非力なハードウェアですが、「**画面表示とマイク・スピーカー I/O だけを担当するシンクライアント**」と割り切れば、驚くほど快適に動きます。重い音声認識や音声合成、LLM 推論はすべて母艦の Mac mini（M1 8GB）側に集約します。

![VoxMate + Echo Show 5 全体アーキテクチャ](/images/echo-show-5-voxmate-assistant/architecture.png)

接続経路を 2 系統に分けるのが運用上のポイントです。

- **Echo Show 5**（USB 常時接続）：`start.sh` が `adb reverse tcp:8000 tcp:8000` を常時維持するため、端末上の Chrome からは `http://localhost:8000` で直接アクセスできます。ブラウザのマイク API（`getUserMedia`）は通常 HTTPS 必須ですが、`localhost` なら自己署名証明書なしで動作します。
- **PC・スマートフォン**（外出先・別室）：Tailscale Serve 経由で HTTPS 公開し、家庭内 LAN や外出先から安全に同じセッションへ接続できます。アラームの停止操作やタイマー、会話履歴も WebSocket で全端末にリアルタイム同期されます。

## 手順1：Echo Show 5 第1世代のアンロックと LineageOS 18.1 導入

Echo Show 5 第1世代（コードネーム `checkers` / MediaTek MT8163）は、一度はファームウェア更新で塞がれたはずのブートローダーアンロック用脆弱性（`fastbrick`）が、後のビルド `Fire OS 6575.1 (NS65751/8851)` で **リバート（巻き戻し）されて再び使えるようになっていた** ことが XDA Developers のスレッドで報告されていました。

:::message alert
**改造・アンロックに関する注意**
ブートローダーアンロックおよびカスタム ROM 導入はメーカー保証の対象外となり、手順を誤ると端末が起動しなくなる（文鎮化する）リスクがあります。本記事は所有端末での個人検証記録であり、実施する場合は自己責任のもと、必ず XDA の一次スレッドで最新手順と対象ビルドを確認してください。なお、アンロック用ツールの再配布は行いません。
:::

### 1-1. 対象ビルドの確認と OTA 更新の遮断

1. Echo Show 5 の「設定」→「デバイスオプション」→「デバイス情報」を開き、システムバージョンが `Fire OS 6575.1 (NS65751/8851)` であることを確認します。
2. 確認できたら **即座に Wi-Fi をオフ** にし、Amazon の自動アップデートで対策済みビルドへ上がらないように遮断します。

### 1-2. Windows 実機でのアンロックと TWRP 導入

MediaTek の BootROM / Fastboot ドライバ周りが安定しているため、アンロック作業は Windows PC で進めるのが確実です。

1. XDA Developers の `checkers` スレッドから `amonet-checkers-v2.0.2` と LineageOS 18.1 イメージ一式を入手し、Windows PC に Google USB Driver を導入します。
2. 端末の電源を完全に落とし、展開した `amonet-checkers-v2.0.2` 内の `fastbrick.bat` を実行します。
3. スクリプトの指示に従ってボリュームボタン操作（または指定の接続タイミング）で USB 接続し、アンロック完了後にそのまま TWRP リカバリを起動します。

### 1-3. LineageOS 18.1 の焼成と USB デバッグ有効化

1. TWRP 画面で「Wipe」→「Advanced Wipe」から Data / Cache を初期化します（MicroUSB / USB 経由で `adb push` できるよう接続を確認）。
2. PC から LineageOS 18.1 の ZIP ファイルを転送してインストールします。

```bash:lineageos-flash.sh
adb push lineage-18.1-*-UNOFFICIAL-checkers.zip /sdcard/
# TWRP の「Install」から上記 ZIP を選択して焼成し、再起動する
```

3. 起動後、初期設定を済ませて「設定」→「端末情報」→「ビルド番号」を 7 回タップして開発者向けオプションを有効にし、**USB デバッグを ON** にします。あとは Chrome を入れておけば端末側の準備は完了です。

詳細なツールと実機報告は XDA Developers の公式スレッドを参照してください。

https://xdaforums.com/t/unlock-root-twrp-unbrick-echo-show-5-checkers.4718281/

## 手順2：母艦（Mac mini / PC）に VoxMate をセットアップする

母艦側では Python 3.12+ と [`uv`](https://docs.astral.sh/uv/)、`ffmpeg`、`jq` を使います（Echo Show 5 を USB 接続する場合は `adb` も必要です）。

### 2-1. リポジトリの取得と依存パッケージの同期

```bash:setup-voxmate.sh
git clone https://github.com/Naruto-Takahashi/voxmate.git
cd voxmate
uv sync
```

`uv sync` を実行すると、Apple Silicon Mac では高速な `mlx-whisper` が、Linux / Windows 環境では `faster-whisper` がプラットフォームに応じて自動インストールされます。

### 2-2. VOICEVOX Engine の準備

- **macOS (Apple Silicon)**: [VOICEVOX Engine Releases](https://github.com/VOICEVOX/voicevox_engine/releases) から `voicevox_engine-macos-arm64-*.tar.gz` を取得し、解凍した中身（`run` 実行ファイル一式）を `voicevox/macos-arm64/` に配置します（`./start.sh` が自動起動します）。
- **Linux / Windows**: Docker 等で `http://127.0.0.1:50021` に待ち受けさせておきます。

```bash:voicevox-docker.sh
docker run --rm -p 50021:50021 voicevox/voicevox_engine:cpu-latest
```

### 2-3. AI バックエンド設定（`.env` の作成）

`VOXMATE_BACKEND` 環境変数で AI バックエンドを切り替えられます。

```bash:backends/__init__.py
_BACKENDS = {
    "agy": PersistentAgySession,
    "claude": ClaudeCodeSession,
    "gemini": GeminiApiSession,
}
```

:::message alert
**AI バックエンド選定と利用規約上の注意**
開発中は手元の Antigravity CLI（`agy`）や Claude Code CLI（`claude`）でも動作検証を行いましたが、Google Antigravity の利用規約（`antigravity.google/terms`）にはサードパーティ製ツールとの接続や自動連携を制限する条項があります。
常時稼働の音声アシスタントとして安全・クリーンに運用する場合は、公式 API キーを使う **Gemini API バックエンド（`VOXMATE_BACKEND=gemini`）** の利用を強く推奨します。
:::

`.env.example` をコピーして `.env` を作成し、Gemini API キーを設定します。

```bash:.env
VOXMATE_BACKEND=gemini
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-2.5-flash

VOICEVOX_URL=http://127.0.0.1:50021
VOICEVOX_SPEAKER=3
```

### 2-4. サーバー起動と Echo Show 5 / スマホ接続

```bash:run-voxmate.sh
# macOS（VOICEVOX Engine + VoxMate + adb reverse を一括維持）
./start.sh

# Linux / Windows（外部で VOICEVOX Engine 起動済みの場合）
uv run python -m uvicorn server:app --host 127.0.0.1 --port 8000
```

- **Echo Show 5 側**: USB で母艦に繋いだ状態で Chrome を開き、`http://localhost:8000` へアクセスします。初回のみマイク権限を許可してください。
- **スマホ・他 PC 側**: Tailscale を使っている場合は以下で HTTPS 公開すると、外出先や別室からもマイク入力付きで同期利用できます。

```bash:tailscale-serve.sh
tailscale serve --bg 8000
```

## 手順3：低遅延ハイブリッド応答とカスタムウェイクワードの構築

基本動作が確認できたら、実運用での待ち時間を削る 2 つの仕組みを導入・調整します。

![低遅延ハイブリッド応答パイプライン](/images/echo-show-5-voxmate-assistant/voice-pipeline.png)

### 3-1. 0 秒台ファストパス（`fast_path.py`）

「今何時？」「3 分後にタイマーかけて」「電気つけて」といった定型操作は LLM に投げず、`fast_path.py` が正規表現とキャッシュ済みデバイス一覧で即座に処理します。ずんだもん（`〜のだ！`）や四国めたん（`〜ですわ！`）などキャラクター口調の語尾もローカル側で整形するため、音声合成の立ち上がりまで 0 秒台に収まります。

### 3-2. openWakeWord で 0.2 秒起動の「ちょっと聞いて」を学習する

初期実装の「無音検知 → Whisper 文字起こし → 先頭ワード判定」では発話終了後 **約 2.5 秒のラグ** があったため、`openWakeWord` を使って全キャラ共通ウェイクワード **「ちょっと聞いて」** 専用の ONNX モデル（`models/chotto_kiite.onnx`、約 460KB）を自作しました。

リポジトリに学習済みモデルを同梱しているためすぐ試せますが、自分の声や別の環境に合わせて再学習したい場合は以下の 3 ステップで実行できます。

```bash:train-kws.sh
# 1. VOICEVOX 全話者から学習用データ（ポジティブ 800件 / ネガティブ 1200件）を生成
uv run python tools/kws_generate.py --pos 800 --neg 1200 --out-dir data/kws

# 2. ブラウザで http://localhost:8000/kws を開き、自分の生声を録音して data/kws に追加する

# 3. M1 Mac / CPU 上の PyTorch で Conv1d モデルを学習し ONNX 化
uv run python tools/kws_train.py --data-dir data/kws --out models/chotto_kiite.onnx --epochs 35
```

この学習済みモデルを `/ws/kws` WebSocket で 80ms ごとに推論させたところ、オーナーの生声テストでポジティブ 25/25 検知・誤検知 0/10、チャイム起動まで **約 0.2 秒以内** を達成できました。

## 手順4：VOICEVOX 音素リップシンクと立ち絵モーションの連携

立ち絵画像を配置しなくてもキャラクターカラーのプレースホルダーで動作しますが、坂本アヒル氏の立ち絵 PSD（各自で規約に従ってダウンロードしたもの）を `custom_psd/` に配置すると、全 12 キャラクター分・各 25 枚の差分 PNG を自動抽出できます。

```bash:extract-portraits.sh
uv run python tools/extract_portraits.py --psd-dir custom_psd --out-dir static/portraits
```

共通バウンディングボックス基準で切り出すため、`<img>` の `src` を切り替えても位置が 1 ピクセルもズレません。

- **LLM 感情タグ**: LLM が出力する `[expr:happy]` や `[expr:shy]` を文ごとにパースし、発話中の表情・腕ポーズ・CSS モーションを切り替えます。
- **音素リップシンク**: VOICEVOX Engine の `audio_query` が返すモーラごとの子音長・母音長から `[開始ms, 口形状]` タイムラインを生成し、Web Audio API の再生時刻と同期して **「あ・い・う・え・お」＋口閉じの 6 状態リップシンク** を動かします。感情表情の発話中も口閉じ差分（`<表情>_c`）へ切り替えることで、腕ポーズや眉がチラつかずに口元だけを自然に動かせます。

## まとめと利用クレジット

古い Echo Show 5 をシンクライアント化し、重い推論を母艦に逃がしつつ「ファストパス」「専用 KWS」「音素リップシンク」を組み合わせることで、実用的な速度と愛着の湧く対話体験を両立できました。

手元に余っているスマートディスプレイや Raspberry Pi がある方は、ぜひ `VOXMATE_BACKEND=gemini` を設定して自分だけの音声アシスタント環境を試してみてください。

:::message
**立ち絵素材・音声合成クレジットとライセンス**
- **立ち絵素材**: 坂本アヒル氏 配布素材（[利用規約・ガイドライン](https://zan-shin.net/guideline/) / [東北ずん子・ずんだもん公式ガイドライン](https://zunko.jp/guideline.html)）。本リポジトリおよび記事内では PSD・PNG 素材の再配布は行っておらず、各配布元の規約に従ってローカル環境に配置する構成としています。
- **音声合成**: [VOICEVOX](https://voicevox.hiroshiba.jp/)
:::

## 参考リンク

https://github.com/Naruto-Takahashi/voxmate

https://xdaforums.com/t/unlock-root-twrp-unbrick-echo-show-5-checkers.4718281/

https://antigravity.google/terms
