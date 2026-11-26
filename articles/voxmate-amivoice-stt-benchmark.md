---
title: "AI秘書の誤認識をAmiVoice APIで撃退！Whisperとガチ比較"
emoji: "🗣️"
type: "tech"
topics: ["amivoice", "whisper", "voicevox", "python", "ai"]
published: false
---

```text
発話       : 明日の朝9時に起こして
mlx-whisper: 明日の朝くじに起こして。
```

最近の LLM は賢いので、前後の文脈から「くじ＝9時」と気を利かせて補完してくれる場面も少なくありません。ただ、文字起こしが毎回こうして別の語へ揺れると、たまに意図しない解釈へ転んだり、何より M1 Mac mini 8GB のローカル推論だけで毎回 **約 1.5 秒** 待たされたりします。

前回の記事で Echo Show 5（第1世代）をカスタム ROM 化し、自宅の Mac mini で動く自作音声アシスタント [VoxMate](https://github.com/Naruto-Takahashi/voxmate) を作りました。

https://zenn.dev/nalt/articles/echo-show-5-voxmate-assistant

ところが実際に使い始めると、手前の文字起こし（ローカルの `mlx-whisper`）が開発用語や時刻を別の言葉に書き換えることがありました。

そこで **AmiVoice API** の同期 HTTP インタフェースと単語登録（`profileWords`）を VoxMate に組み込み、ローカル Whisper と 4 条件・16 文で比較しました。この記事を読むと、**Python から AmiVoice API の `profileWords` を正しく渡す実装** と、**実測データに基づくローカル STT とクラウド STT の使い分け** が分かります。

## VoxMate に `VOXMATE_STT=amivoice` を組み込む

VoxMate は入力音声を `ffmpeg` で 16kHz モノラル WAV に正規化してから認識器へ渡しているため、同期 HTTP エンドポイント（`https://acp-api.amivoice.com/v1/nolog/recognize`）へのクライアントを 1 ファイル足すだけで切り替えられます。

```python:stt_amivoice.py
from pathlib import Path
from urllib.parse import quote
import httpx

DEFAULT_AMIVOICE_URL = "https://acp-api.amivoice.com/v1/nolog/recognize"

def build_amivoice_d_param(engine: str = "-a-general", profile_words: str = "") -> str:
    parts = [f"grammarFileNames={engine.strip() or '-a-general'}"]
    words = profile_words.strip()
    if words:
        # d パラメータは半角スペース区切りのため、profileWords 全体は URL エンコードする
        parts.append(f"profileWords={quote(words, safe='')}")
    return " ".join(parts)

def transcribe_wav_amivoice(wav_path: Path, app_key: str, profile_words: str = "") -> str:
    d_param = build_amivoice_d_param(profile_words=profile_words)
    with httpx.Client(timeout=30.0) as client:
        resp = client.post(
            DEFAULT_AMIVOICE_URL,
            files={
                "u": (None, app_key),
                "d": (None, d_param),
                "a": (wav_path.name, wav_path.read_bytes(), "audio/wav"),
            },
        )
        resp.raise_for_status()
        payload = resp.json()
        if payload.get("code"):
            raise RuntimeError(f"AmiVoice error ({payload['code']}): {payload.get('message')}")
        return str(payload.get("text") or "").strip()
```

`.env` に API キーと登録したい語彙を並べるだけで有効化できます。

```bash:.env
VOXMATE_STT=amivoice
AMIVOICE_APP_KEY=your_api_key_here
AMIVOICE_ENGINE=-a-general
AMIVOICE_PROFILE_WORDS=プルリク ぷるりく|Claude くろーど|Nix にっくす|GitHub ぎっとはぶ|Issue いしゅー
```

### 実装時に踏んだ 2 つの罠

1. **`profileWords` の書式違いはエラーにならず無視される**
   最初は `表記|読み` を半角スペースで並べてしまい、エラーも出ないまま素の認識結果が返り続けました。公式仕様は「**表記（半角スペース）ひらがな読み**」を `|` で繋ぎ、さらに `d=` へ渡す前に URL エンコードする形式です。
2. **アカウント登録直後の認証エラー**
   ACP マイページでクーポンを適用し、カード登録を終えた直後に叩くと `received illegal service authorization` になりました。マイページの注意書きにある「使用可能になるまで 10 分程度」を待ってから再試行すると通りました。

## 空耳コレクション（実測ログから抜粋）

開発の日常語・ツール名・コマンド・一般会話を含む 16 文を VOICEVOX の複数話者（男女 5 名）で生成し、各エンジンへ通した実測結果から、挙動の差が出たケースを影響度の低い順に並べます。

### ケース 01：Claude

> 発話：「Claudeに聞いてみて」

| 条件 | 認識結果（そのまま LLM へ渡る文字列） |
| :--- | :--- |
| `mlx-whisper`（素） | `グロードに聞いてみて。` |
| `mlx-whisper`（プロンプト有） | `Grodeに聞いてみて。` |
| `AmiVoice`（素） | `Claudeに聞いてみて` |
| `AmiVoice`（`profileWords` 有） | `Claudeに聞いてみて` |

Whisper は `initial_prompt` に `Claude` を含めても `Grode` に寄せたのに対し、AmiVoice は素の `-a-general` の時点で正しく `Claude` を返しました。

### ケース 02：Issue

> 発話：「GitHubのIssueを見せて」

| 条件 | 認識結果（そのまま LLM へ渡る文字列） |
| :--- | :--- |
| `mlx-whisper`（素） | `GitHubの一種を見せて、` |
| `mlx-whisper`（プロンプト有） | `GitHubのIssueを見せて` |
| `AmiVoice`（素） | `GitHubのイシューを見せて` |
| `AmiVoice`（`profileWords` 有） | `GitHubのIssueを見せて` |

音声合成では `GitHubの一種` に、筆者の肉声で試すと何度話しかけても `GitHubの一周` に変換されました。LLM が文脈で補える場合もありますが、そのまま検索クエリへ渡す処理では確実にコケます。

### ケース 03：プルリク・リバート

> 発話：「プルリクをマージして」「直前のコミットをリバートして」

| 条件 | プルリク（発話：プルリクをマージして） | リバート（発話：直前のコミットをリバートして） |
| :--- | :--- | :--- |
| `mlx-whisper`（素） | `ブルリクをマージして。` | `直前のコミットをリバートして、` |
| `AmiVoice`（素） | `ぷるリップをマージして、` | `直前のコミットをリバーとして、` |
| `AmiVoice`（`profileWords` 有） | `プルリクをマージして` | `直前のコミットをリバートして、` |

`ぷるリップ` でも `リバーとして` でも、辞書にない語が近い音の日本語へ引き寄せられています。

### ケース 04：朝の目覚まし（一般会話のつもりだった文）

> 発話：「明日の朝9時に起こして」

| 条件 | 認識結果（そのまま LLM へ渡る文字列） |
| :--- | :--- |
| `mlx-whisper`（素） | `明日の朝くじに起こして。` |
| `mlx-whisper`（プロンプト有） | `明日の朝くじに起こして。` |
| `AmiVoice`（素） | `明日の朝9時に起こして` |
| `AmiVoice`（`profileWords` 有） | `明日の朝9時に起こして` |

固有名詞の検証用に用意した文ではなく、基準線として入れた普通の日本語で差が出ました。アラーム登録に「9時」ではなく「くじ」が渡ると、朝起きられなくなります。

### ケース 05：Slack に返信

> 発話：「Slackに返信しといて」

| 条件 | 認識結果（そのまま LLM へ渡る文字列） |
| :--- | :--- |
| `mlx-whisper`（素） | `スラックに変身しといて。` |
| `mlx-whisper`（プロンプト有） | `スラックに変身しといて` |
| `AmiVoice`（素） | `Slackに返信しといて` |
| `AmiVoice`（`profileWords` 有） | `Slackに変身しといて` |

興味深いことに、`AmiVoice`（素）は文脈から `Slackに返信しといて` と正しく変換できましたが、他の 3 条件では `変身` に引っ張られました。

### 全条件で正解だった文（対照）

もちろん、すべてがボケるわけではありません。「本番にデプロイして」「新しいブランチを切って」「古いテーブルをドロップして」「CIが落ちてる」「今日の天気は？」の 5 文は、4 条件すべてで意味が通る正しい文字列を返しました。

## 誰も悪くない。ただ辞書がなかっただけ

エンジン側は音響特徴からもっともらしい音素列を出しただけで、辞書に載っていない単語を聞き間違えるのは自然な挙動です。`profileWords` はその場でドメイン辞書を 1 行差し込めるため、モデルを再学習することなく誤認識の大部分を潰せます。

## 4 条件 × 16 文 × 3 回反復の定量比較

全 16 文を各条件 3 回ずつ実行（初回ウォームアップ除外）し、句読点・空白を除いた文字誤り率（CER）、対象キーワード正解率、推論レイテンシ中央値をまとめました。

![VoxMate 音声認識 4条件比較（精度 × 速度）](/images/voxmate-amivoice-stt-benchmark/stt-benchmark.png)

| 比較条件 | 全体 CER | 対象文 CER | 一般文 CER | キーワード正解率 | レイテンシ中央値 |
| :--- | :---: | :---: | :---: | :---: | :---: |
| `mlx-whisper`（素） | 19.59% | 21.09% | 9.09% | 58.3% (14/24) | 1,490.7 ms |
| `mlx-whisper`（プロンプト有） | 7.49% | 7.26% | 9.09% | **87.5%** (21/24) | 1,503.9 ms |
| `AmiVoice`（素） | 14.37% | 16.42% | **0.00%** | 62.5% (15/24) | **952.3 ms** |
| `AmiVoice`（`profileWords` 有） | **3.97%** | **4.54%** | **0.00%** | **87.5%** (21/24) | 1,023.5 ms |

- **精度面**：`AmiVoice`（`profileWords` 有）が全体 CER **3.97%**、一般文 CER **0.00%** で最も安定しました。
- **速度面**：M1 Mac mini のローカル `mlx-whisper`（約 1.5 秒）に対し、AmiVoice API は通信込みで **約 0.95〜1.02 秒** と約 1.5 倍高速でした。
- **限界と使い分け**：本計測は VOICEVOX 合成音声による検証であり、実環境の雑音・残響下やオフライン環境では挙動が変わります。また `git push` を `きっとプッシュ` と拾うような母音・子音の揺れは、読みを複数パターン登録する工夫が有効です。

## まとめ

ローカル Whisper は通信不要で無料という強みがある一方、固有名詞の誤変換や M1 環境での 1.5 秒前後の待ち時間がネックになります。AmiVoice API と `profileWords` を組み合わせることで、**文字誤り率を 19.59% から 3.97% まで下げつつ、応答テンポも短縮** できました。

Echo Show 5 などの音声アシスタントを普段使いのテンポに近づけたい場合は、`VOXMATE_STT=amivoice` での運用を試してみてください。

https://github.com/Naruto-Takahashi/voxmate

https://docs.amivoice.com/amivoice-api/manual/
