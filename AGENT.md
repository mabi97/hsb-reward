# HSB Reward Agent

## Project Overview

Dashboard tĩnh (static) hiển thị dữ liệu reward/loyalty từ Snowflake.
- Dữ liệu được export từ Snowflake **1 lần/ngày** dưới dạng JSON
- Frontend thuần **HTML + JavaScript** (không cần framework)
- Deploy dưới dạng static site (GitHub Pages / S3 / Netlify / server đơn giản)

## Architecture

```
Snowflake
   │
   │ (daily export — script/dbt/query)
   ▼
JSON data files (data/*.json)
   │
   │ (fetch / load)
   ▼
HTML + JS Dashboard (index.html, assets/)
   │
   │ (static hosting)
   ▼
Browser
```

## Tech Stack

| Layer | Công nghệ |
|-------|-----------|
| Data source | Snowflake (read-only, daily snapshot) |
| Data format | JSON files tĩnh |
| Frontend | HTML5 + Vanilla JS (hoặc Chart.js / Echarts cho chart) |
| Styling | CSS thuần hoặc lightweight lib (e.g. Pico CSS) |
| Deploy | Static hosting (TBD) |

## Data Flow

1. **Export** — Query Snowflake, output ra `data/*.json`
2. **Commit / Upload** — Đẩy file JSON lên hosting (hoặc CI/CD tự động)
3. **Dashboard** — `index.html` fetch JSON và render chart/bảng

## File Structure (dự kiến)

```
hsb_reward/
├── AGENT.md
├── index.html          # Entry point
├── assets/
│   ├── main.js         # Logic chính
│   └── style.css
├── data/
│   └── *.json          # Data export từ Snowflake
└── scripts/
    └── export.py       # Script export Snowflake → JSON (chạy daily)
```

## Implementation Notes

- Không cần backend — mọi thứ là file tĩnh
- Data JSON được fetch trực tiếp bằng `fetch()` hoặc import
- Nếu cần filter/aggregate nhẹ thì xử lý ở JS phía client
- Chart library: ưu tiên **Chart.js** (nhỏ gọn) hoặc **ECharts** (phong phú hơn)

## Open Questions

- [ ] Dashboard hiển thị những metric nào? (doanh số, điểm thưởng, số lượng thành viên, v.v.)
- [ ] Deploy lên đâu? (GitHub Pages, S3, server nội bộ?)
- [ ] Script export Snowflake đã có chưa hay cần viết mới?
- [ ] Có yêu cầu về xác thực (login) không hay public?
