# RentScope — 租金實價登錄查詢平台 架構設計

> MVP: 雙北租賃實價查詢 + 地圖呈現 \
> 動機: 政府租賃實價資料已開放, 但官方查詢體驗差、資料分散 → 做一個好用的版本

## 1. 技術選型

| 項目 | 選擇 | 理由 |
|---|---|---|
| 後端 | Python + FastAPI | async 原生支援、Pydantic 驗證、自帶 OpenAPI 文件 |
| 前端 | Next.js (App Router) + TypeScript | SSR/ISR 兼顧 SEO 與資料更新, strict mode 保證型別安全 |
| UI | Bootstrap | 元件成熟, 不重造輪子 |
| 資料庫 | PostgreSQL 15 + PostGIS | 地理查詢是核心需求, GiST 索引支援地圖範圍搜尋 |
| 地圖 | MapLibre GL + OpenStreetMap | 開源、零成本, 避免商用地圖 API 帳單風險 |
| 部署 | GCP e2-micro + Nginx + Docker Compose | 永久免費層, dev/prod 環境同構 |
| CI/CD | GitHub Actions | 與 repo 整合最順, 免費額度夠用 |

## 2. 系統架構

```
內政部資料供應系統 (每月 1、16 日批次 CSV)
        │
        ▼
  ETL Worker (Python, APScheduler/cron)
   fetch → validate → clean → geocode → upsert
        │
        ▼
  PostgreSQL + PostGIS ◄── FastAPI (REST /api/v1)
                                ▲
                          Next.js (SSR + MapLibre 地圖)
                                ▲
                          Nginx reverse proxy (TLS, gzip, rate limit)
                                ▲
                          Docker Compose @ GCP e2-micro (Ubuntu LTS)
```

- 資料來源: [內政部不動產成交案件資料供應系統](https://plvr.land.moi.gov.tw/Index) open data, 租賃案件 CSV, 免費不限流量; 不爬第三方網站, 無法律風險
- 地圖: MapLibre GL + OpenStreetMap tiles (零成本); 地理編碼用 TGOS 或 Nominatim, 結果快取進 DB
- 範圍: 先台北市 + 新北市兩個縣市的租賃檔

## 3. 資料模型 (草案)

```sql
districts        -- 行政區 (city, district, geometry)
buildings_types  -- 建物型態 lookup
rental_records   -- 核心事實表
  id, source_id (去重鍵), city, district, address_masked,
  building_type, total_area_m2, floor, total_floor,
  rent_price, rent_per_m2, lease_date,
  has_furniture, has_manager,
  location GEOMETRY(Point, 4326),  -- PostGIS
  raw JSONB,                        -- 保留原始列, 利於重跑清洗
  created_at, updated_at
etl_runs         -- 每次批次的 audit log (成功/失敗/筆數)
```

索引: `(city, district, lease_date)` 複合索引、`location` GiST 索引、`source_id` unique (冪等 upsert)

## 4. 後端 — Clean Architecture

```
backend/
  src/
    domain/          # Entities + value objects, 零外部依賴
      models.py      # RentalRecord, District, Money, Area
    application/     # Use cases, 只依賴 domain + repository 介面
      use_cases/     # SearchRentals, GetDistrictStats
      interfaces/    # RentalRepository (Protocol)
    infrastructure/  # SQLAlchemy repo 實作、geocoder client
      db/            # models (ORM), repositories, migrations (Alembic)
    api/             # FastAPI routers + Pydantic schemas (DTO)
      v1/            # /rentals, /districts, /stats
  etl/               # 獨立 pipeline: fetch → clean → load
  tests/
    unit/            # domain + use cases (不碰 DB)
    integration/     # repo + API (testcontainers 起真 PostgreSQL)
```

Design patterns:

- **Repository pattern** — use case 只依賴 `RentalRepository` Protocol, DB 可替換、單元測試可 mock
- **Dependency Injection** — FastAPI `Depends` 組裝, composition root 在 api 層
- **DTO 分離** — Pydantic schema (API 邊界) ≠ domain model ≠ ORM model, 各層變更互不污染
- **Pipeline pattern (ETL)** — 每個 stage 純函式化, 可獨立測試; 失敗記錄進 `etl_runs`, 支援重跑 (冪等)

## 5. 前端 — Next.js

```
frontend/
  app/
    page.tsx              # 首頁: 搜尋 + 地圖 (SSR)
    districts/[id]/       # 行政區行情頁 (SSG/ISR — SEO)
  components/             # SearchFilter, RentalMap, RecordTable, PriceBadge
  lib/api.ts              # typed API client
  e2e/                    # Playwright
```

- TypeScript strict mode; UI 用 Bootstrap
- 地圖 marker 聚合 (supercluster), 行政區頁用 ISR 兼顧 SEO 與資料更新

## 6. 測試策略

| 層 | 工具 | 內容 |
|---|---|---|
| Unit | pytest | domain 邏輯、use cases (mock repo)、ETL 清洗函式 |
| Integration | pytest + testcontainers | repository 對真 PostgreSQL、API contract |
| E2E | Playwright | 搜尋流程、地圖互動、關鍵頁面 smoke test |
| Frontend | Vitest + Testing Library | 元件邏輯 |

## 7. CI/CD (GitHub Actions)

```
PR:   ruff + mypy + eslint → unit tests → integration tests → build
main: 上述全部 → docker build & push (GHCR) → SSH deploy 到 VM
      → docker compose pull && up -d → smoke test → 失敗自動 rollback
```

## 8. 部署與運維

- GCP e2-micro (永久免費層) + Ubuntu LTS, Nginx 直接裝在主機上管理 TLS 與流量
- Nginx: TLS (Let's Encrypt/certbot)、gzip、rate limiting、reverse proxy 到 frontend:3000 / backend:8000
- ETL 用 systemd timer 或 container 內 scheduler, 每月 1、16 日 + 隔日補跑
- 監控從簡: healthcheck endpoint + UptimeRobot; 日誌 docker logs + logrotate

## 9. 安全

- SQL injection: 全程 SQLAlchemy 參數化
- Input validation: Pydantic 邊界驗證 + 查詢參數白名單
- Rate limiting: Nginx `limit_req` + API 層 slowapi
- Headers: CSP、HSTS、X-Content-Type-Options (Nginx 統一加)
- 個資: 僅使用政府已去識別化資料, 地址只到路段
- Secrets: 環境變數注入, 不進 git; CI 用 GitHub Secrets

## 10. Roadmap

1. **M1 資料通** — ETL 抓雙北租賃檔入庫, schema + migration 完成
2. **M2 API** — 搜尋/統計 endpoints + unit/integration tests
3. **M3 前端** — 搜尋頁 + 地圖 + 行政區頁
4. **M4 上線** — CI/CD 全綠、Playwright E2E、部署 + TLS
5. **M5 延伸** — 行情統計 dashboard、租金趨勢圖、訂閱通知
