# TODO — YouBike 台北市車況儀表板

> 由淺入深的開發清單。細節見 `系統分析與設計.md`。
> 開工日 2026-09-04,交件 2026-09-16(約 12 天)。
> 進度落後時**從下往上砍**:先砍階段 7,再砍階段 6 的進階項,確保階段 0–5 完成。

## 退路階梯(卡住就退一階,專案仍完整可交)

- [ ] **第 1 階(目標)**:雲端 PostgreSQL + GitHub Actions 排程 + 後端 API + 完整前端 + 部署
- [ ] **第 2 階**:砍掉所有進階分析(潮汐/分群/天氣),只留 MVP 五個功能
- [ ] **第 3 階**:不用雲端 DB,GitHub Actions 跑完把 SQLite 檔 commit 回 repo,後端讀 repo 內的檔
- [ ] **第 4 階(最低)**:純本地,展示時在自己電腦開前端 + `uvicorn`

---

## 階段 0 — 環境與資料探索(先能看到資料)

- [ ] 建虛擬環境(**建在家目錄,不能建在 `/Volumes/LUKECHENG` 外接碟**):`python3 -m venv ~/.venvs/youbike-dashboard`
- [ ] 寫 `requirements.txt`,`pip install -r requirements.txt`
- [ ] 寫 `.gitignore`(已有,確認含 `.venv/`、`*.db`、`.env`)、`.env.example`
- [ ] 用 curl / Python 打一次官方 API,記錄真實欄位名到 `系統分析與設計.md`
- [ ] 寫 `config.py`(集中設定:API URL、DATABASE_URL、抓取間隔)

## 階段 1 — 資料管線本機跑通(MVP 基礎)

- [ ] `ingest/fetcher.py`:`requests` + `timeout` + `tenacity` 重試,回傳原始 list
- [ ] `ingest/transform.py`:官方欄位 → 內部欄位、去站名前綴、型別轉換、過濾停用站與壞座標、時間轉 `Asia/Taipei`
- [ ] `db/models.py`:`Station`、`Snapshot`、`StationHourly` 三張表(`Snapshot` 有 `UNIQUE(sno, ts)`)
- [ ] `db/session.py`:依 `DATABASE_URL` 切換 SQLite / PostgreSQL
- [ ] `db/init_db.py`:`python -m db.init_db` 在本機 SQLite 建出 3 張表
- [ ] `ingest/store.py`:`station` 做 upsert;`snapshot` 做 insert-or-ignore
- [ ] `ingest/run_once.py`:串 fetch → transform → store,印出「這次抓到幾站、新增幾列」
- [ ] **驗收**:`python -m ingest.run_once` 連跑兩次,`snapshot` 不出現重複 `(sno, ts)`

## 階段 2 — 雲端排程開始累積(⚠️ 卡交件,越早越好)

- [ ] 開 Neon 免費 PostgreSQL,拿到連線字串
- [ ] 對雲端 DB 跑一次 `python -m db.init_db` 建表
- [ ] `.github/workflows/ingest.yml`:cron `*/10 * * * *` → 執行 `run_once`
- [ ] `DATABASE_URL` 放進 repo 的 Settings → Secrets
- [ ] **驗收**:push 後 Actions 每 10 分鐘成功執行,雲端 DB 每次增加約 1,800 列 snapshot
- [ ] (備援)`scheduler.py`:APScheduler 本地每 N 分鐘跑 `run_once`

## 階段 3 — 後端 API 骨架(MVP)

- [ ] `app/main.py`:FastAPI 實例 + CORS 設定
- [ ] `app/schemas.py`:Pydantic 回傳模型
- [ ] `app/deps.py`:注入 DB session
- [ ] `app/routers/stations.py`:`GET /stations/live`(讀最新 snapshot)
- [ ] `GET /health` 回 200
- [ ] **驗收**:本機 `uvicorn`,`/docs` 打得開,`/stations/live` 回全站 JSON

## 階段 4 — 前端即時畫面(MVP,面試主秀)

- [ ] 找一份台北市 12 行政區 GeoJSON 放 `frontend/geo/`
- [ ] `frontend/index.html` + `style.css`:頂部 KPI 卡列、左地圖、右圖表區
- [ ] `js/config.js`(API base URL、官方 JSON URL、刷新間隔)、`js/api.js`(fetch 包裝 + 錯誤處理)
- [ ] `js/map.js`:ECharts `registerMap` + geo 底圖 + scatter 站點,顏色=滿位率、大小=容量、hover tooltip
- [ ] `js/kpi.js`:全市可借車總數、啟用站數、缺車站數、爆滿站數
- [ ] `js/main.js`:`setInterval` 每 60 秒刷新地圖與 KPI;`visibilitychange` 時暫停
- [ ] **驗收**:本機開網頁看到台北地圖 + 上千彩色站點,60 秒自動更新

## 階段 5 — 部署上線(拿到公開網址)

- [ ] 後端上 Render / Railway,設 `DATABASE_URL`、`CORS_ORIGINS`
- [ ] 前端上 Vercel / GitHub Pages,`config.js` 指向線上 API
- [ ] 修部署問題:CORS、HTTPS mixed content、免費方案冷啟動、geojson 路徑
- [ ] **驗收**:有一個公開網址能看到會動的即時地圖

## 階段 6 — 歷史分析與圖表(MVP 分析功能)

- [ ] `analysis/aggregate.py`:`snapshot` → `station_hourly`(每小時平均可借車、缺車/爆滿比例)
- [ ] `.github/workflows/aggregate.yml`:每小時跑一次聚合(或併進 ingest)
- [ ] `analysis/peak.py` + `GET /stats/peak-heatmap`:小時 × 行政區 缺車率矩陣
- [ ] `frontend/js/heatmap.js`:ECharts 熱力圖,看得出上下班尖峰帶
- [ ] `analysis/ranking.py` + `GET /stats/ranking` + `frontend/js/ranking.js`:最常缺車 / 爆滿 Top 10
- [ ] `GET /stations/{sno}/history` + `frontend/js/station-detail.js`:點站點顯示近 7 天折線
- [ ] `app/cache.py`:TTL 記憶體快取,套用到 stats endpoints

## 階段 7 — 進階(有時間才做,擇一)

- [ ] **通勤潮汐** `analysis/flow.py` + `GET /stats/commute-flow` + 前端:相鄰快照 sbi 差 → 淨流入/流出站分類
- [ ] **站點分群** `analysis/cluster.py` + `GET /stats/clusters` + 前端:日內曲線 K-means 分住宅/辦公/觀光型
- [ ] **天氣關聯**:接中央氣象署 API,分析降雨 vs 使用量

## 階段 8 — 收尾與交件

- [ ] `README.md`:定位、架構圖、畫面截圖、套件、本地執行步驟、已知限制、未來方向
- [ ] `analysis.md`:帶數字的分析結論(潮汐、缺車站排行、分群結果)
- [ ] `tests/`:`test_transform.py`、`test_store.py`、`test_api.py`
- [ ] `.github/workflows/ci.yml`:每次 push 跑 ruff + pytest
- [ ] 錄一段 2 分鐘 demo 影片或截圖組
- [ ] 全流程手動驗證一遍,線上版 / repo / 文件三者一致
