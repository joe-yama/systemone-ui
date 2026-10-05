# SystemOne UI

English | [日本語](README.ja.md)

A minimal local UI for Ollama's SystemOne decision models (clef-flash 9B, clef 27B, etc.).
These are not chat models: you send a state and typed questions to `/v1/systemone` and get back a probability for each option.

A single `index.html`, no build step, no dependencies. UI in Japanese and English.

![Screenshot](example-screenshot.png)

## Requirements

- Ollama 0.35.1 or later
- A decision model: `ollama pull clef-flash` (or `clef`)

## Getting started

```sh
ollama serve   # skip if already running
```

Serve this folder and open http://localhost:8000:

```sh
# Python
python3 -m http.server 8000 --bind 127.0.0.1

# uv (no Python install needed)
uv run --no-project python -m http.server 8000 --bind 127.0.0.1

# Docker
docker compose up -d --build     # stop: docker compose down
```

Open the page over `http://`, not `file://` (Ollama rejects `Origin: null`). With Docker the browser still talks to Ollama directly, so keep `http://127.0.0.1:11434`.

Remote Ollama: on that host, set `OLLAMA_HOST=0.0.0.0` and `OLLAMA_ORIGINS=http://localhost:8000`, then enter its URL in the UI.

## Question types

- `choice`: pick one of the keyed options. Returns per-option probabilities and `confidence`.
- `score`: ordered criteria, indexed 0, 1, 2… Returns the expected index (`score`), probabilities, and `confidence`.
- `noul`: yes/no. Returns only p(true) (`noul`), no confidence.

## Notes

- Only pulled models with the `decision` capability are listed.
- State: "Text" sends a string; "JSON" sends the parsed object/array.
- The last 20 successful runs are kept in `localStorage`. Clicking one restores it without resending.
- Latency is measured in the browser and includes model load time, so the first run can take several seconds.
- Not supported: image input (`images`) and extra parameters such as `keep_alive`.

## API

```jsonc
// POST /v1/systemone
{
  "model": "clef-flash",
  "state": "text, or a JSON object/array",
  "questions": {
    "team":     { "type": "choice", "instructions": "...", "criteria": { "billing": "...", "technical": "..." } },
    "urgent":   { "type": "noul",   "instructions": "...", "criteria": { "true": "...", "false": "..." } },
    "severity": { "type": "score",  "instructions": "...", "criteria": ["low", "medium", "high"] }
  }
}
```

Docs: [ollama.com/library/clef-flash](https://ollama.com/library/clef-flash)

## License

[MIT](LICENSE)
