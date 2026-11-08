# Qwen LINE Translator

A FastAPI-based LINE bot that translates text using a local Ollama model. Chinese messages are translated into English; other languages are translated into Taiwan Traditional Chinese. The default model is `qwen3.5:9b`.

For example, sending `早安` requests an English translation. The model is instructed to preserve names, URLs, numbers, mentions, hashtags, line breaks, and emoji, but accuracy and exact formatting are not guaranteed.

## Branches

| Branch | Purpose |
| --- | --- |
| `qwen-line-translator` | This local translation bot |
| `main` | LINE VOOM image analysis with Gemini, market reports, and Notion integration |

Run `git switch qwen-line-translator` before following this README. Commit or save work before switching. The two versions require different dependencies and credentials; switching branches does not reconfigure or restart a running bot.

This branch does not call Gemini or Notion. The repository name reflects its original reporting workflow.

## Architecture

`LINE event → signature verification → language detection → memory cache → Ollama /api/chat → split output → LINE reply`

Each model request includes a fixed translation instruction and the current message only. Previous conversation messages are not sent. Identical source text and target-language pairs share a process-local cache; both source text and translations remain in memory until eviction or process exit. No database or persistent conversation store is implemented.

| Endpoint | Behavior |
| --- | --- |
| `GET /` | Service information and configured model name |
| `GET /health` | Checks nonempty LINE settings, Ollama connectivity, and model presence |
| `POST /callback` | Verifies the LINE signature and handles text-message events |

## Requirements

- Python 3.10 or later; the documented verification environment used Python 3.11.5.
- Ollama with the configured model installed. No minimum Ollama version, model inference benchmark, or hardware requirement has been validated for this repository.
- A LINE Messaging API channel and an accessible HTTPS webhook endpoint for live messaging.

Direct dependencies are pinned in `requirements.txt`; transitive dependencies are not locked. Unit tests mock model responses and require neither live LINE credentials nor a running model.

## Installation

From the repository root, using Windows PowerShell:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

Do not overwrite an existing `.env`; reconcile its settings manually. Replace the sample `LINE_CHANNEL_ACCESS_TOKEN` and `LINE_CHANNEL_SECRET` values with your test channel credentials. The sample values are placeholders, not working credentials.

On macOS/Linux, use `.venv/bin/python` instead of `.\.venv\Scripts\python.exe` and `cp .env.example .env` to create the configuration file.

## Start locally

Ensure Ollama is running. If the desktop application already provides the service, do not start a second instance. Otherwise run `ollama serve` in another terminal.

```powershell
ollama list
# Download only if the model is not already installed.
ollama pull qwen3.5:9b
.\.venv\Scripts\python.exe app.py
```

Model download size and inference speed depend on the model and hardware. This repository makes no throughput or latency claim.

Open `http://127.0.0.1:5000/`, or check readiness:

```powershell
Invoke-RestMethod http://127.0.0.1:5000/health
```

The health endpoint returns 200 only when LINE settings are nonempty, Ollama responds, and the configured model is listed; otherwise it returns 503. It does not authenticate LINE credentials or perform inference.

`python app.py` binds to `0.0.0.0:5000`, allowing traffic on available network interfaces. For local-only development:

```powershell
.\.venv\Scripts\python.exe -m uvicorn app:app --host 127.0.0.1 --port 5000
```

## Connect LINE

Forward a public HTTPS endpoint to local port 5000. For example, after installing ngrok separately:

```powershell
ngrok http 5000
```

Set the LINE channel's webhook URL to `https://example.com/callback`, replacing the example domain with your endpoint. Enable webhook delivery and review automatic reply settings to avoid duplicate responses. One channel has one webhook destination; use separate channels to run the translator and report bot independently.

Messages still travel through LINE. Translation is local only when `OLLAMA_BASE_URL` points to your local machine; configuring a remote host sends source text to that host.

## Configuration

`python-dotenv` loads `.env`, while existing process environment variables take precedence. All settings appear in `.env.example`.

| Variable | Default | Purpose |
| --- | --- | --- |
| `LINE_CHANNEL_ACCESS_TOKEN` | Empty | LINE reply credential |
| `LINE_CHANNEL_SECRET` | Empty | Webhook signature verification |
| `OLLAMA_BASE_URL` | `http://127.0.0.1:11434` | Ollama endpoint |
| `OLLAMA_MODEL` | `qwen3.5:9b` | Model name |
| `OLLAMA_TIMEOUT_SECONDS` | `120` | Translation timeout in seconds; minimum 0.1 |
| `OLLAMA_KEEP_ALIVE` | `10m` | How long Ollama keeps the model loaded |
| `OLLAMA_THINK` | `false` | Model thinking switch; `1`, `true`, `yes`, and `on` enable it |
| `MAX_INPUT_CHARS` | `5000` | Input limit after trimming surrounding whitespace |
| `LINE_MESSAGE_CHARS` | `4900` | Output characters per segment; clamped to 64–4900 |
| `LINE_MAX_REPLY_MESSAGES` | `5` | Maximum segments per reply; clamped to 1–5 |
| `CACHE_TTL_SECONDS` | `1800` | Cache entry lifetime in seconds |
| `CACHE_MAX_SIZE` | `512` | Maximum cache entries |
| `LOG_LEVEL` | `INFO` | Python logging level, such as `INFO` or `WARNING` |
| `PYTHON_DOTENV_DISABLED` | Empty | Skip `.env` loading when set to `1`, `true`, `yes`, or `on` |

Invalid numeric strings fall back to defaults. Integer settings have a minimum of 1 unless stated otherwise. Non-finite timeout values are rejected. `.env.example` explicitly sets `PYTHON_DOTENV_DISABLED=0`; set it in the process environment before import when running isolated tests.

## Repository layout

```text
app.py              Configuration, detection, translation, cache, and HTTP endpoints
tests/test_app.py   Offline unittest coverage
requirements.txt    Direct dependency versions
.env.example        Placeholder configuration
docs/REVIEW.md      Review scope, verification evidence, and remaining limitations
```

The service remains a single module because the workflow is small. There is no additional database, queue, or hosted model dependency.

## Tests and checks

The following commands disable private `.env` loading. They do not send translation requests to LINE or Ollama:

```powershell
$env:PYTHON_DOTENV_DISABLED = '1'
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
.\.venv\Scripts\python.exe -m compileall -q app.py tests
.\.venv\Scripts\python.exe -m pip check
Remove-Item Env:PYTHON_DOTENV_DISABLED
```

Coverage includes translation direction, mocked request construction, cache reuse, output splitting, empty input, malformed model responses, and invalid timeout values. Eight tests passed in the documented verification environment. A separate clean temporary environment was installed from `requirements.txt`, ran the tests, and served `GET /` with HTTP 200.

There is no separate build artifact or configured lint tool. Compilation checks syntax; it is not comprehensive static analysis. Live model quality, LINE delivery, load, and production deployment have not been validated.

## Limitations and operational considerations

- Inference completes before the webhook response. Slow generation may exceed LINE webhook or reply-token timing limits. There is no durable queue, event deduplication, or application-level request throttling.
- Language detection combines heuristics with `langdetect`. Short or mixed-language messages and Japanese text containing only Han characters can be misclassified. Target-language selection cannot be overridden per message.
- Only text events are translated. Long output is truncated after the configured reply count. Splitting does not account for every Unicode length-counting edge case.
- The prompt asks the model to treat user messages as text rather than instructions, but this does not guarantee resistance to instruction-like input or incorrect output.
- Cached content is shared across users within the process. It is not a conversation history, but it is still temporary retention of message content.
- Application logs do not deliberately include full message text. Exceptions can include network addresses or execution details; keep private logs out of demonstrations and public repositories.
- Public production use needs further work on access control, resource limits, concurrency, monitoring, and retention requirements. Local startup is not proof of production readiness.

The user reported replacing previously exposed credentials, and remote branch history was cleaned. Copies held elsewhere or old GitHub caches may still exist. Never distribute the original uncleaned backups. See `docs/REVIEW.md` for the review scope; that historical record may remain in Chinese.

No project license is included. Public visibility does not establish permission to reuse the code under an unspecified license.
