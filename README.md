# LINE Bot Local - 本地開源模型智慧助理

本專案將 LINE 官方帳號（Messaging API）直接連接至本機運行的 Ollama 開源模型（Google Gemma 4）。
整個 Bot 僅由一支 Python 程式組成，透過 Cloudflare 臨時通道接收 LINE Webhook 訊息，具備即時聯網搜尋回答、解析行程排入 Google 日曆、辨識照片並儲存至 Google 雲端硬碟，以及將互動紀錄同步寫入 Google 試算表等完整功能。

---

## 🛠️ 環境需求與安裝指南

本專案之 Python 程式僅依賴 Python 內建標準函式庫，**完全不需要 `pip install` 任何第三方套件**。

### 1. 安裝 Python (3.10 以上，建議 3.13)
```powershell
winget install --id Python.Python.3.13 -e --accept-package-agreements --accept-source-agreements
```

### 2. 安裝 Ollama 並下載 Gemma 4 模型
```powershell
winget install --id Ollama.Ollama -e --accept-package-agreements --accept-source-agreements --silent
# 確保 Ollama 在背景運行後，下載 Gemma 4 (e4b) 模型：
ollama pull gemma4:e4b
```
> 若顯示卡記憶體較小（如 4GB 以下或純 CPU 運行），亦可下載輕量版 `ollama pull gemma4:e2b`。

### 3. 安裝 Cloudflared（通道工具）
用於提供安全的暫時性 HTTPS 網址，讓 LINE 伺服器能將 Webhook 事件推送至本機：
```powershell
winget install --id Cloudflare.cloudflared -e --accept-package-agreements --accept-source-agreements
```

### 4. （選用）安裝 Node.js 與 Google Workspace CLI (`gws`)
用於 Google 日曆、雲端硬碟、試算表功能：
```powershell
winget install --id OpenJS.NodeJS.LTS -e --accept-package-agreements --accept-source-agreements --silent
npm install -g --allow-scripts=@googleworkspace/cli @googleworkspace/cli
gws auth login -s "drive,sheets,calendar"
```

---

## 🚀 如何運行 (How to Run)

1. 開啟 PowerShell 或終端機，進入專案目錄：
   ```powershell
   cd line-bot-local
   ```

2. 啟動 Bot：
   ```powershell
   python line_bot_local.py
   ```

3. 程式啟動時會主動詢問三項金鑰／ID（**輸入時畫面隱藏或不寫入檔案，確保金鑰零外洩風險**）：
   - **LINE Channel Access Token**（長效通行碼）
   - **Channel Secret**（LINE Developers → Basic settings）
   - **Google 試算表 ID**（若不寫試算表可直接按 Enter 略過）

4. 程式會自動啟動本機伺服器、透過 `cloudflared` 建立通道，並呼叫 LINE API 將 Webhook 自動更新為通道網址。

5. **結束運行**：在終端機按下 `Ctrl+C`，程式會在關閉前自動呼叫 API 將 LINE 的 Webhook 網址還原回原本設定（如先前的 Apps Script 網址）。

---

## 💡 我改了什麼 (Modifications)

1. **模型選擇**：
   - 預設採用 **`gemma4:e4b`**（多模態開源大語言模型），兼具繁體中文語意理解、推論能力與多模態圖片識別能力。

2. **新增即時聯網搜尋與網頁讀取容錯降級機制 (`fallback_search` & `fallback_fetch`)**：
   - **問題背景**：原版程式依賴 Ollama 原生實驗性端點 `/api/experimental/web_search` 與 `/api/experimental/web_fetch`，但在本機未綁定登入 Ollama 官方雲端帳號時，會遭遇 `HTTP 401 Unauthorized` 錯誤，導致模型無法順利連網查詢最新即時資訊。
   - **改良實作**：在程式中新增純標準函式庫實現的自動降級備用搜尋與網頁純文字抓取機制。當 Ollama 線上搜尋端點受阻或未授權時，程式會自動無縫切換至備用搜尋引擎，即時爬取前幾筆最相關結果與內文反饋給 Gemma 4，使 Bot 無需綁定任何付費或外部 API 即可 100% 穩定回答即時新聞與事實問題，並附上完整來源連結。

3. **強化 Windows 環境下的相容性路徑偵測**：
   - 針對 Windows 系統下 `cloudflared` 與 `gws.exe` 安裝於 Program Files 或 AppData 時可能未即時載入 PATH 的狀況，強化自動尋找路徑邏輯，避免因環境變數延遲而誤報找不到執行檔。
