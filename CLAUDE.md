# CLAUDE.md

> 給任何一個新的 Claude Code 對話看的專案交接文件。開工前請先讀這份,再讀下方指定的文件。

## 專案一句話

台灣 YouBike(台北市)公共自行車**即時車況儀表板 + 歷史分析**:
每 10 分鐘抓官方即時 JSON,累積成時間序列,分析通勤潮汐與尖離峰缺車,用 ECharts 呈現,並公開部署。
用途:學校期末專案 + 面試作品集(涵蓋資料分析 / 資料工程 / 前端視覺化)。

## 一定要先讀的文件

| 文件 | 內容 |
|---|---|
| `系統分析與設計.md` | **主文件**。完整架構、技術選型、資料庫設計、每個檔案的職責、API 設計、關鍵實作方法、每日開發計畫、風險降級方案 |
| `主題選擇與市場分析.md` | 為什麼選 YouBike(比較過的其他題目、面試三方向考量、降級階梯)。避免給出走回頭路的建議 |
| `爬蟲資料視覺化專案規劃.md` | 最初的技術構想筆記(背景參考,已被上面兩份取代) |

## 技術棧摘要(細節見系統分析與設計.md 第四章)

- **語言**:Python 3.11+
- **抓取/清洗**:requests、tenacity、pandas
- **資料庫**:本地開發 SQLite;正式環境免費雲端 PostgreSQL(Neon/Supabase)。用 SQLAlchemy 抽象層切換
- **排程**:GitHub Actions cron `*/10 * * * *`(主力);APScheduler(本地備援)
- **API**:FastAPI + uvicorn + Pydantic
- **前端**:純 HTML/CSS/JS + ECharts v5(不用框架)
- **部署**:後端 Render/Railway、前端 Vercel/GitHub Pages、資料庫 Neon/Supabase

## 開發慣例

- 虛擬環境:`python -m venv .venv && source .venv/bin/activate`
- **不要 commit / 不要讓 Claude 讀**:`.venv/`、`__pycache__/`、`*.db`、`*.sqlite`、`.env`、爬蟲累積的大型資料檔
- 祕密(`DATABASE_URL`、API key)只放 `.env`(本地)與部署平台的環境變數面板,絕不進 git
- 時間一律用 `Asia/Taipei` 時區
- snapshot 寫入必須 idempotent(`ON CONFLICT DO NOTHING`,靠 `UNIQUE(sno, ts)`)

## 目前進度

> 每完成一個階段就更新這一段,讓下一個對話知道從哪接手。

- 狀態:**規劃完成,尚未開始寫程式**
- 已完成:
  - 系統分析與設計文件
  - 主題選擇與市場分析文件
- 下一步(見系統分析與設計.md 第十章「第 0 週」):
  1. 建 repo、venv、`requirements.txt`、`.gitignore`、`config.py`、`.env.example`
  2. **用 curl/瀏覽器打一次官方 API 確認真實欄位名**(附錄有指令)
  3. 寫 `db/models.py`、`db/session.py`、`db/init_db.py`,建出三張表
- 待辦(非程式):
  - 找一份台北市 12 行政區 GeoJSON,放 `frontend/geo/`
  - 開 Neon 或 Supabase 免費 PostgreSQL

## 重要決策紀錄

- **題目選 YouBike 而非 104 求職分析**:YouBike 有官方開放 API、零反爬,20 天內可如期完成;即時地圖展示效果好;DE(高頻排程、去重、回補)與前端(即時地圖、熱力圖)兩個面試方向特別加分。104 反爬強、GitHub 同類專案過多。
- **即時地圖讓前端直接打官方 JSON**:官方是 Azure Blob 靜態檔、更新頻繁,直連即時性最好,且不依賴會休眠的免費後端。後端保留 `/stations/live` 代理當備援。
- **主力排程用 GitHub Actions 而非部署後端內的排程**:免費後端(Render 等)會休眠,排程會停;Actions cron 不會休眠,是可靠的歷史累積來源。
- **資料庫**:本地 SQLite、正式 PostgreSQL,用 SQLAlchemy 讓兩者可切換;因為 Actions 與 API 要共用同一份持續增長的資料。

## 交件期限

2026/09/16
