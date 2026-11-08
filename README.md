# LINE VOOM Report Bot

A FastAPI-based LINE bot that downloads images from LINE VOOM posts, uses Gemini to generate morning or post-market reports, and saves the results to Notion. It can enrich extracted facts with Taiwan market data from FinMind.

This is a locally operated reporting tool, not a hosted service. Report prompts and user-facing messages may remain in Traditional Chinese; the documentation language does not affect model instructions.

## Branches

| Branch | Purpose |
| --- | --- |
| `main` | VOOM image analysis and morning/post-market reports |
| `qwen-line-translator` | Local Ollama/Qwen text translation; see that branch's README |

Use `git switch main` for this version. Commit or save changes before switching branches. Each version has different dependencies and configuration; switching Git branches does not restart a running bot or update its environment.

## How it works

1. LINE sends a signed text event to `POST /callback`.
2. The bot reads the report-mode prefix and the first URL, acknowledges the request, and normally starts a background thread.
3. A Playwright Chromium subprocess opens the VOOM image viewer and downloads images into `voom_images/`.
4. Gemini extracts facts from the images. Optional FinMind context is appended before a second Gemini call composes the report.
5. The bot creates a Notion child page and sends its URL through LINE.

For example, `2 https://voom.line.me/post/EXAMPLE_POST_ID` requests a post-market report. Prefix `1` selects a morning report; omitting the prefix also selects morning mode. The post ID here is a placeholder, not a working demonstration.

## Requirements

- Python 3.10 or later; syntax checks were performed with Python 3.11.5. Dependencies use features incompatible with the former Python 3.8 claim.
- A desktop session for the downloader's visible Chromium browser.
- A LINE Messaging API channel, a Gemini API key, and a Notion integration with access to the destination pages.
- A public HTTPS endpoint forwarding to the local bot.
- A local `prompts.py` module, described below. It is deliberately ignored by Git and is not included in a fresh clone.

Direct dependency versions are pinned in `requirements.txt`; transitive dependencies are not locked. The configured Gemini model must be available to your account. The source default is documented below, not a guarantee of provider availability.

## Installation

Run these commands from the repository root on the `main` branch. Windows PowerShell examples:

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
.\.venv\Scripts\python.exe -m playwright install chromium
```

On macOS/Linux, use `.venv/bin/python` in place of `.\.venv\Scripts\python.exe`. A graphical browser session is still required; headless server deployment is not configured.

### Create the local prompt module

Create `prompts.py` next to `app.py`. The application imports three string variables from it and replaces `{image_labels}` and `{facts_table}` placeholders. A fresh clone cannot import `app.py` until this module exists.

The following is a minimal example, not the original private prompts or a validated report template:

```python
extract_facts_prompt = """Read the supplied images and extract factual statements.
Preserve source dates and distinguish observations from opinions.
Do not invent unreadable values. Identify uncertainty.
Images:
{image_labels}
"""

morning_report_prompt = """Write a concise Traditional Chinese morning market brief.
Use only the facts below. Preserve dates and flag missing or conflicting data.
Do not present speculation as fact or make personalized investment recommendations.
Facts:
{facts_table}
"""

after_hours_report_prompt = """Write a concise Traditional Chinese post-market brief.
Use only the facts below. Preserve dates and flag missing or conflicting data.
Do not present speculation as fact or make personalized investment recommendations.
Facts:
{facts_table}
"""
```

You can write these strings in Chinese to control the report style. Do not place credentials or private source material in reusable prompts.

### Configure credentials and destination pages

Create `.env` in the repository root. This branch does not include an `.env.example`; use the placeholder-only template below. Do not overwrite an existing private configuration.

```dotenv
LINE_CHANNEL_ACCESS_TOKEN=replace_with_your_test_channel_access_token
LINE_CHANNEL_SECRET=replace_with_your_test_channel_secret
GOOGLE_API_KEY=replace_with_your_gemini_api_key
NOTION_TOKEN=replace_with_your_notion_integration_token
NOTION_PARENT_PAGE_MORNING_URL=https://www.notion.so/00000000000000000000000000000000
NOTION_PARENT_PAGE_AFTER_HOURS_URL=https://www.notion.so/00000000000000000000000000000000

# Source-code default; choose a model supported by your account.
GEMINI_VISION_MODEL=gemini-3-flash-preview

# Disable external market-data requests while checking local setup.
MARKET_CONTEXT_ENABLED=false
```

The all-zero page IDs are intentionally invalid placeholders. Replace each with a page URL or ID and grant your integration access to that page. Set both mode-specific page settings if you use both report modes. `NOTION_PARENT_PAGE_URL` is **not** read by the current implementation and is not a fallback.

`python-dotenv` loads `.env`; existing process environment variables take precedence. Keep `.env`, downloaded images, prompts, and execution logs private.

## Start the bot

```powershell
.\.venv\Scripts\python.exe -m uvicorn app:app --host 127.0.0.1 --port 5000
```

Alternatively, `python app.py` starts on `0.0.0.0:5000`, exposing the service on available network interfaces. Run it from the project root because image and downloader paths are relative to the working directory.

Forward HTTPS traffic to port 5000 using your chosen tunnel or reverse proxy. For example, after separately installing ngrok:

```powershell
ngrok http 5000
```

Set the channel's webhook URL to `https://example.com/callback`, replacing the example domain with your endpoint, and enable webhook delivery in LINE Developers Console. Review automatic reply settings to avoid duplicate responses. One channel has one webhook destination; use separate channels to run this bot and the translator independently.

This branch has no application health endpoint. A running web server alone does not verify LINE credentials, browser automation, model access, or Notion permissions.

## Configuration reference

Required credentials and page destinations are listed above. Additional settings are read directly from the source:

| Variable | Default | Purpose |
| --- | --- | --- |
| `GEMINI_VISION_MODEL` | `gemini-3-flash-preview` | Model used for extraction and synthesis |
| `GEMINI_TIMEOUT_SECONDS` | `300` | Per-call image-analysis timeout, seconds |
| `GEMINI_SYNTHESIS_TIMEOUT_SECONDS` | `max(GEMINI_TIMEOUT_SECONDS, 600)` | Text synthesis timeout, seconds |
| `GEMINI_IMAGE_BATCH_SIZE` | `3` | Images per batch when the batch path is selected; minimum 1 |
| `GEMINI_DIRECT_BATCH_MIN_IMAGES` | `0` | Positive threshold to select batching directly; 0 disables that shortcut |
| `GEMINI_RATE_LIMIT_MAX_CALLS` | `5` | Process-local call-start limit per window; minimum 1 |
| `GEMINI_RATE_LIMIT_WINDOW_SECONDS` | `60` | Limiter window, seconds; minimum 1 |
| `GEMINI_RATE_LIMIT_SAFETY_SECONDS` | `0.25` | Extra wait margin; minimum 0 |
| `MAX_VOOM_IMAGES` | Unset | Maximum loaded images sent to analysis; use a positive integer |
| `MARKET_CONTEXT_ENABLED` | `true` | Fetch FinMind context; set `false` to disable |
| `FINMIND_API_TOKEN` | Unset | Optional token; otherwise anonymous requests are used |
| `FINMIND_TIMEOUT_SECONDS` | `20` | Market-data request timeout, seconds |
| `FINMIND_LOOKBACK_DAYS` | `45` | Query lookback, with a minimum of 25 days |
| `FINMIND_FOCUS_SECURITIES` | `0050,0056,00878,00919,00981A,00631L,00632R,2330,2317,2454` | Comma-separated context symbols, not recommendations |
| `FINMIND_FOCUS_LIMIT` | Number of configured symbols | Maximum focus symbols; minimum 1 |

The downloader has a fixed 180-second subprocess timeout. `MAX_VOOM_IMAGES` limits analysis after downloading; it does not cap browser downloads. Application-side rate limiting is not a guarantee of provider quota compliance.

## Repository layout

```text
app.py              FastAPI webhook, report pipeline, Gemini and Notion integration
market_data.py      FinMind queries and market-context formatting
voom_downloader.py  Visible-browser image downloader, run as a subprocess
requirements.txt   Direct dependency versions
prompts.py          Required local prompt strings; not tracked
.env                Local credentials and configuration; not tracked
voom_images/        Downloaded image files; not tracked
```

The downloader can also be run separately. It opens a browser and accesses the supplied post:

```powershell
.\.venv\Scripts\python.exe voom_downloader.py 'https://voom.line.me/post/EXAMPLE_POST_ID'
```

Replace the placeholder with a post you are authorized to access.

## Verification

This branch has no committed automated test suite or configured lint tool. A credential-free syntax check is:

```powershell
.\.venv\Scripts\python.exe -m py_compile app.py market_data.py voom_downloader.py
.\.venv\Scripts\python.exe -m pip check
```

The syntax check was run successfully on the restored morning-report source. It does not import dependencies, validate private prompts, or exercise external services. The separate translator branch's eight passing unit tests do not verify this branch. A complete clean installation and live LINE/VOOM/Gemini/FinMind/Notion round trip have not been revalidated during this documentation update.

## Design trade-offs and limitations

- Background work runs in daemon threads, without a durable queue, event deduplication, or restart recovery. A process exit can interrupt a report.
- Jobs share `voom_images/`, which is cleared before each download. Concurrent jobs can overwrite or delete each other's files; use one request at a time. Multiple workers do not isolate this directory.
- Browser automation depends on VOOM page selectors and public accessibility. Login requirements, layout changes, or closing the visible browser can interrupt downloads.
- Gemini output may be inaccurate. Keep dates, source attribution, missing data, and uncertainty visible; manually review reports before relying on them.
- FinMind context can be incomplete or unavailable. Different index definitions and dates are not directly comparable; the pipeline includes guidance to preserve those distinctions.
- The webhook currently logs the raw request body, and downloader/error logs can expose URLs and identifiers. Do not publish logs or expose this personal tool to untrusted traffic without additional hardening.
- URL validation, concurrency control, access restrictions, retention, and operational monitoring require further work before public production deployment.
- Notion conversion supports only a subset of Markdown. LINE delivery remains subject to channel permissions, messaging limits, and token validity.
- External API calls and LINE messaging may consume quotas or incur charges. No cost, throughput, or reliability benchmark is claimed.

Credentials previously committed to history were reported replaced, and remote branch history was cleaned. This does not remove third-party copies or guarantee that old GitHub caches have disappeared. Do not distribute original uncleaned backups.

No project license is included. Public visibility should not be treated as permission to reuse the code under an unspecified license.
