# SystemOne UI

Ollama の SystemOne 系判断モデル (**clef-flash** 9B、**clef** 27B など) を試すための、ローカルで動く簡易 UI です。
これらはチャットモデルではありません。`/v1/systemone` エンドポイントに「状態」と「型付きの質問」を渡すと、選択肢ごとの確率が返ります。

画面は `index.html` 1 ファイルだけで動きます (ビルド不要・外部依存なし)。日本語と英語を切り替えられます。

| ファイル | 内容 |
|---|---|
| `index.html` | UI 本体 (HTML/CSS/JS) |
| `Dockerfile` / `compose.yaml` | Docker で配信する場合に使用 |

## 必要なもの

- Ollama **0.35.1 以上** (clef 系に対応したバージョン)
- 判断モデルを 1 つ以上: `ollama pull clef-flash` (必要なら `ollama pull clef`)
- 配信手段のいずれか: Python 3 / uv / Docker

## 起動手順

まず Ollama を起動します (起動済みなら不要です)。

```sh
ollama serve
```

次に、このフォルダで次のいずれかを実行し、ブラウザで **http://localhost:8000** を開きます。

```sh
# Python
python3 -m http.server 8000 --bind 127.0.0.1

# uv (Python が無くても可)
uv run --no-project python -m http.server 8000 --bind 127.0.0.1

# Docker
docker compose up -d --build     # 停止: docker compose down
```

`index.html` をダブルクリックして `file://` で開くと、ブラウザが `Origin: null` を送るため Ollama に拒否されます。必ず `http://localhost` 経由で開いてください。
`http://localhost:*` と `http://127.0.0.1:*` からのアクセスは Ollama の既定の CORS 設定で許可されているので、プロキシは不要です。
Docker で動かしても、Ollama に接続するのはブラウザです。そのため、接続先 URL は `http://127.0.0.1:11434` のままで構いません。

## 使い方

1. **言語**: ヘッダー右上の「日本語 / English」で切り替えます (選択は保存されます)。
2. **接続**: 接続先 URL (既定 `http://127.0.0.1:11434`) を入力します。ヘッダーに接続状態と Ollama のバージョンが表示されます。
3. **モデル**: 取得済みのモデルのうち、判断モデル (`/api/tags` の `capabilities` に `decision` を含むもの) だけが一覧に出ます。clef-flash / clef 以外の SystemOne 系モデルも、取得すれば自動で表示されます。
4. **プリセット**: ボタン 1 つで状態と質問が埋まります。表示言語に応じて、日本語版か英語版が読み込まれます。
   - 問い合わせのルーティング (choice)
   - バグ報告の分類 (choice + noul、状態は JSON)
   - レビューの満足度 (score + noul)
5. **状態**: 「テキスト」は入力をそのまま文字列として送ります。「JSON」はパースして、オブジェクトまたは配列として `state` に入れます。
6. **質問スキーマ**: フォームで質問を追加・削除します。「JSON」タブでは `questions` オブジェクトを直接編集できます (JSON が不正な間はフォームに戻れません)。
   - `choice`: 選択肢の「キー」と「説明」 (2〜26 個)
   - `score`: 順序付きの基準 (2〜26 段階。上から 0, 1, 2…)
   - `noul`: 真偽の説明 (任意。両方空なら省略)
7. **判定する**: 結果が質問ごとの横棒グラフで表示されます。
   - `choice`: 確率の高い順に並び、信頼度 (`confidence`) を表示
   - `score`: 基準の順序のまま並び、期待値 (`score`) と最も近い基準、信頼度を表示
   - `noul`: 「真」「偽 (1−p)」の 2 本。API が信頼度を返さないため「なし」と表示
   - あわせて、応答時間 (ブラウザで計測した ms)、モデル名、`input_tokens` を表示
8. **送信リクエスト (curl)**: 実際に送る内容を curl コマンドの形で確認・コピーできます。
9. **履歴**: 成功した実行を、直近 20 件まで `localStorage` に保存します。クリックすると入力と結果を復元します (再送信はしません)。接続先 URL、選択モデル、言語も保存されます。

## API の形 (参考)

```jsonc
// POST /v1/systemone
{
  "model": "clef-flash",
  "state": "テキスト、または JSON オブジェクト/配列",
  "questions": {
    "team":     { "type": "choice", "instructions": "...", "criteria": { "billing": "説明", "technical": "説明" } },
    "urgent":   { "type": "noul",   "instructions": "...", "criteria": { "true": "...", "false": "..." } },
    "severity": { "type": "score",  "instructions": "...", "criteria": ["低", "中", "高"] }
  }
}
// レスポンス: answers.<キー> に
//   choice → choice / probabilities / confidence
//   noul   → noul (真である確率)
//   score  → score (期待値) / legend / probabilities / confidence
```

詳細は https://ollama.com/library/clef-flash を参照してください。

## 既知の制限

- **画像入力には未対応**です。API は `images` で base64 の PNG/JPEG/WebP を受け付けますが、この UI からは送れません。
- `keep_alive` などの追加パラメーターは指定できません。
- モデルの一覧は、`capabilities` が `decision` を含むかどうかで判定しています。Ollama 0.35.1 より前のバージョンでは、判断モデルが一覧に出ません。
- 応答時間はブラウザ側で計測した値で、通信時間とモデルの読み込み時間を含みます。初回はモデルの読み込みに数秒以上かかります。
- `localhost` 以外のホストで動く Ollama に接続する場合は、そのホスト側で `OLLAMA_ORIGINS` に、この UI のオリジン (例: `http://localhost:8000`) を追加する必要があります。
- 履歴はブラウザごとの `localStorage` に保存されます。プライベートウィンドウなどでは保存されないことがあります。
- 送信前の検証 (キーの形式、選択肢の数など) は UI 側で独自に行っているものです。最終的に受け付けるかどうかは、サーバーの応答に従います。
- UI の文言は日本語と英語を切り替えられますが、エラー時に表示するサーバーの応答メッセージは英語のままです。
