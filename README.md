# Qwen LINE 訊息翻譯 Bot

透過 LINE 接收文字，交給本機 Ollama（執行語言模型的服務）翻譯，再回覆 LINE。
中文翻成英文；其他語言翻成台灣繁體中文。例如「早安」會以英文回覆。
模型會被要求保留名稱、網址、換行和 emoji，但不保證翻譯完全正確。

目前 `main` 為 Qwen 翻譯 Bot，已整合原 `codex/qwen-line-translator` 功能分支。
倉庫名稱保留早期 VOOM／Gemini／Notion 名稱；晨報流程仍可從歷史提交查看，
目前版本不包含該功能，也不呼叫 Gemini 或 Notion。

## 環境與安裝

- Python 3.10 以上；本次驗證版本為 Python 3.11.5。
- 直接依賴固定於 `requirements.txt`；間接依賴未鎖定。
- Ollama 與 `qwen3.5:9b`。本次未實測模型推論，未驗證 Ollama 最低版本或硬體效能。
- 真實收發需 LINE Messaging API（程式收發 LINE 訊息的介面）Channel，以及可連入的 HTTPS 網址。

在專案根目錄使用 Windows PowerShell：

```powershell
python -m venv .venv
.\.venv\Scripts\python.exe -m pip install -r requirements.txt
Copy-Item .env.example .env
```

編輯 `.env`，填入自己測試 Channel 的 `LINE_CHANNEL_ACCESS_TOKEN` 與 `LINE_CHANNEL_SECRET`。
範例只有虛構佔位值。若已有 `.env`，手動核對設定，不要覆寫。不要分享私人設定或真實訊息。

先確認 Ollama 正在執行；桌面版已啟動時不需再啟動服務。否則在另一個終端執行 `ollama serve`。

```powershell
ollama list
# 僅在尚未下載時執行；會下載模型。
ollama pull qwen3.5:9b
.\.venv\Scripts\python.exe app.py
```

開啟 `http://127.0.0.1:5000/`，或檢查服務：

```powershell
Invoke-RestMethod http://127.0.0.1:5000/health
```

`/health` 在 LINE 設定非空、Ollama 可連線且模型存在時回傳 200，否則回傳 503。
這不會驗證 LINE 憑證有效性，也不會實際翻譯。

## LINE 設定

Webhook（LINE 收到事件後主動通知本服務的網址）為 `https://example.com/callback`，
請換成自己的 HTTPS 網域。開發期間可自行安裝 ngrok 並執行 `ngrok http 5000`。
在 LINE Developers Console 設定 URL、啟用 Webhook，並依需要停用自動回覆。
同一 Channel 只能指定一個 Webhook；與既有 Bot 同時使用時請另建測試 Channel。

預設綁定 `0.0.0.0:5000`，允許網路介面連入。只需本機存取時可用：

```powershell
.\.venv\Scripts\python.exe -m uvicorn app:app --host 127.0.0.1 --port 5000
```

## 設定

`python-dotenv` 讀取 `.env`；已存在的環境變數優先。

| 變數 | 預設值 | 用途 |
| --- | --- | --- |
| `LINE_CHANNEL_ACCESS_TOKEN` | 空 | LINE 回覆憑證 |
| `LINE_CHANNEL_SECRET` | 空 | 事件簽章驗證 |
| `OLLAMA_BASE_URL` | `http://127.0.0.1:11434` | Ollama 位址；若改為遠端，訊息會送往該主機 |
| `OLLAMA_MODEL` | `qwen3.5:9b` | 模型名稱 |
| `OLLAMA_TIMEOUT_SECONDS` | `120` | 翻譯等待秒數，最小 0.1 |
| `OLLAMA_KEEP_ALIVE` | `10m` | 模型保留於記憶體時間 |
| `OLLAMA_THINK` | `false` | 思考開關；1/true/yes/on 視為真 |
| `MAX_INPUT_CHARS` | `5000` | 去除首尾空白後的輸入上限 |
| `LINE_MESSAGE_CHARS` | `4900` | 每段字元數，限制為 64–4900 |
| `LINE_MAX_REPLY_MESSAGES` | `5` | 段數，限制為 1–5 |
| `CACHE_TTL_SECONDS` | `1800` | 快取有效秒數 |
| `CACHE_MAX_SIZE` | `512` | 快取筆數上限 |
| `LOG_LEVEL` | `INFO` | Python 日誌層級，如 INFO、WARNING |
| `PYTHON_DOTENV_DISABLED` | 空 | 1/true/yes/on 時不讀取 `.env`，適合隔離測試 |

數字無法解析時使用預設值；整數下限為 1（每段長度為 64）。逾時設定拒絕無限大及非數值。

## 結構與流程

```text
app.py              設定、語言辨識、翻譯、快取與 HTTP 入口
tests/test_app.py   不連外的 unittest 測試
requirements.txt    直接依賴版本
.env.example        設定範例
docs/REVIEW.md      檢查依據、公開範圍與待處理事項
```

`POST /callback` 驗證 LINE 簽章 → 辨識語言 → 查快取 → Ollama `/api/chat` → 分段 → 回覆 LINE。
每次只有固定翻譯指示與當前訊息，不讀取前後對話。
相同文字與目標語言共用程序內快取；原文及翻譯短暫保留在記憶體，重啟即消失，沒有資料庫。
`GET /` 提供服務資訊；`GET /health` 檢查設定與模型清單。

## 測試與檢查

測試模擬模型回應，不需要真實 LINE 或 Ollama。避免讀取私人 `.env`：

```powershell
$env:PYTHON_DOTENV_DISABLED = '1'
.\.venv\Scripts\python.exe -m unittest discover -s tests -v
.\.venv\Scripts\python.exe -m compileall -q app.py tests
.\.venv\Scripts\python.exe -m pip check
Remove-Item Env:PYTHON_DOTENV_DISABLED
```

涵蓋語言方向、快取、分段與部分錯誤情境。沒有建置產物或已設定的 lint（程式風格及常見問題檢查）工具；
編譯檢查不等於完整靜態分析。

## 設計取捨與限制

- 小型服務採單檔，無額外資料庫或背景工作依賴。
- 同步等待模型再回覆；慢速推論可能超過 LINE 等待或回覆期限。尚無佇列、事件去重、重送控制或流量限制。
- 短句、混合語言、只有漢字的日文可能判錯；不支援手動指定翻譯方向。
- 只處理文字事件；過長輸出截斷。分段未涵蓋所有 Unicode 計數差異。
- 訊息仍經 LINE 傳輸，並送往設定的 Ollama 主機；不持久儲存對話。
- 日誌不主動記錄訊息全文，但例外可能包含網路位址。請勿公開私人日誌。
- 未驗證真實模型品質、端到端收發、壓力或正式部署，不代表適合直接上線。
- 本分支未附授權條款。公開可閱讀不等於授權重用；需由權利人選擇授權。

歷史曾包含憑證；使用者已回報更換，遠端分支歷史也已清理。檢查範圍、驗證結果及限制見 `docs/REVIEW.md`。
