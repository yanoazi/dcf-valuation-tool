# 智能 DCF 估值系統

這是一個高度自動化的財報數據分析與 DCF (現金流折現) 估值系統。它結合了 n8n 的自動化流程、yfinance 的強大數據獲取能力，以及一個動態的前端介面，旨在最小化手動數據輸入，實現快速、穩健的個股估值分析。

## 核心功能

*   **全自動數據獲取**: 只需輸入股票代碼，系統即可透過 n8n 工作流，利用 `yfinance` 獲取最新的 TTM (過去十二個月) 財務數據。
*   **智能容錯與降級**: Python 腳本內建了穩健的數據獲取邏輯，例如在 FCF 或其他關鍵數據缺失時，會自動採取備用計算方案。
*   **TTM 數據優先**: 所有損益表和現金流量表數據均以 TTM 為基礎進行計算，確保估值模型反映最新的公司營運狀況。
*   **動態前端介面**: 一個乾淨、響應式的單頁應用，提供「估值計算」和「財報更新」兩種模式，並具備智能提示功能。
*   **自動化工作表管理**: 當輸入的股票代碼在 Google Sheet 中不存在時，系統會自動複製「範例」工作表來建立新公司的表格，實現無縫擴展。
*   **即時財報更新提醒**: 系統會自動判斷財報是否可能已過期，並主動提示使用者更新數據，確保分析的時效性。

## 專案架構

本系統由四個核心組件構成，各司其職，透過 API 和 Webhook 進行通訊：
1.  **前端應用 (Frontend)**: 使用者與系統互動的唯一入口。負責發起「估值計算」和「財報更新」的請求。
2.  **Google Apps Script (GAS)**: 作為一個輕量級後端，處理前端的計算請求、讀取 Google Sheet 中的數據，並執行 DCF 核心公式。
3.  **n8n 自動化工作流**: 系統的「數據引擎」。由前端的 Webhook 觸發，執行一個強大的 Python 腳本來獲取所有財務數據，然後將結果逐一寫入 Google Sheet。
4.  **Google Sheets**: 作為我們的核心數據庫，儲存所有由 n8n 獲取的原始數據，並作為 GAS 計算的數據來源。

## 安裝與設定

要完整部署此系統，您需要完成以下步驟：

### 1. Google Sheets 設定
- 建立一個新的 Google Sheet 檔案。
- 建立一個名為「**範例**」的工作表，並按照專案截圖的格式，在 A 欄填入所有的 `Key` (e.g., `ticker`, `market_cap`, `wacc` etc.)。這是後續自動建立新工作表的模板。

### 2. Google Apps Script (GAS) 部署
- 在您的 Google Sheet 中，透過 `擴充功能 > Apps Script` 進入編輯器。
- 將本專案提供的最終版 `GAS.gs` 程式碼**完整貼上**。
- 點擊「**部署 > 新增部署作業**」。
- 選擇類型為「**網頁應用程式**」。
- 在「誰可以存取」的選項中，選擇「**任何人**」。
- 點擊「部署」，並複製產生的**網頁應用程式網址**。這個 URL 將用於前端設定。

### 3. n8n 工作流設定 (on Zeabur)
- **Dockerfile**: 確保您用於部署 n8n 的 `Dockerfile` 包含了安裝 Python 函式庫的指令：
  ```dockerfile
  FROM n8nio/n8n:latest
  USER root
  RUN apk add --no-cache python3 py3-pip && \
      pip install --upgrade pip --break-system-packages && \
      pip install yfinance pandas python-dateutil --break-system-packages
  USER node
  ```
- **工作流**:
    1.  建立一個新的工作流，以 `Webhook` 節點開始。複製其 `Test URL`。
    2.  在其後連接一個 `Edit Fields` 節點，用於提取 `ticker`。
    3.  連接一個 `Execute Command` 節點，將本專案最終版的 Python 腳本指令**完整貼上**。
    4.  連接一個 `Loop Over Items` 節點，用於將 Python 輸出的 JSON 轉換為可迭代的項目列表。
    5.  在迴圈中，連接一個 `Google Sheets` 節點，設定其 `Update Row` 操作，並將 `Key` 和 `Value` 與迴圈中的 `$json.key` 和 `$json.value` 對應。
    6.  在迴圈結束 (`done` 分支) 後，連接一個 `Set` 節點，用於產生最終的成功訊息。
- **獲取 Webhook URL**: 儲存並啟用您的 n8n 工作流，將 Webhook 節點的 `Production URL` 複製下來。

### 4. 前端設定
- 將本專案最終版的 `index.html` 程式碼儲存為一個檔案。
- 打開該檔案，在頂部的 `<script>` 區塊中，找到以下兩個常數：
  ```javascript
  const N8N_WEBHOOK_URL = "YOUR_N8N_WEBHOOK_URL";
  const GOOGLE_APPS_SCRIPT_URL = "YOUR_GAS_URL";
  ```
- 將 `YOUR_N8N_WEBHOOK_URL` 和 `YOUR_GAS_URL` 替換為您在步驟 2 和 3 中複製的網址。
- 將此 `index.html` 檔案部署到任何靜態網站託管服務（如 GitHub Pages, Netlify, Vercel, 或 Zeabur）。

## 使用流程

1.  打開前端應用的網址。
2.  在輸入框中輸入您想查詢的股票代碼，然後點擊「查詢」。
3.  **如果數據存在**: 系統會自動帶入預設的成長率和 WACC，您可以直接修改或點擊「計算估值」。
4.  **如果數據不存在/為空**: 系統會引導您前往「更新財報」頁面。
5.  點擊「更新財報」將觸發 n8n 工作流，系統會顯示進度。成功後，您可以返回估值頁面進行計算。
