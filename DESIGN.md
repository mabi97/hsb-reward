# HSB Reward Dashboard — Design System

> Brand reference: [hsb.co.id](https://www.hsb.co.id/)
> Tone: Professional · Trustworthy · Modern Fintech

---

## 1. Color Palette

### Brand Colors

| Token | Hex | Dùng cho |
|-------|-----|----------|
| `--color-gold` | `#F5A623` | Primary CTA, highlight, accent badge |
| `--color-gold-dark` | `#D4881A` | Hover state của gold |
| `--color-green` | `#00A651` | Positive value, tăng, active status |
| `--color-red` | `#E53935` | Negative value, giảm, warning |

### Neutral / Background

| Token | Hex | Dùng cho |
|-------|-----|----------|
| `--color-bg` | `#F8F9FA` | Background trang chính |
| `--color-surface` | `#FFFFFF` | Card, bảng, panel |
| `--color-surface-dark` | `#1A1D2E` | Header, sidebar (dark section) |
| `--color-border` | `#E2E6EA` | Đường viền card, divider |
| `--color-text` | `#1A1A1A` | Text chính |
| `--color-text-secondary` | `#6C757D` | Label, caption, phụ |
| `--color-text-inverse` | `#FFFFFF` | Text trên nền tối |

### Semantic

| Token | Hex | Dùng cho |
|-------|-----|----------|
| `--color-success` | `#00A651` | Hoàn thành, đạt target |
| `--color-warning` | `#FFC107` | Cảnh báo nhẹ |
| `--color-danger` | `#E53935` | Lỗi, vượt ngưỡng |
| `--color-info` | `#1565C0` | Thông tin trung tính |

---

## 2. Typography

Font ưu tiên: **Inter** (Google Fonts) — modern, dễ đọc, chuẩn fintech.
Fallback: `system-ui, -apple-system, sans-serif`

```html
<link href="https://fonts.googleapis.com/css2?family=Inter:wght@400;500;600;700&display=swap" rel="stylesheet">
```

### Scale

| Role | Size | Weight | Dùng cho |
|------|------|--------|----------|
| Page Title | 24px | 700 | Tiêu đề trang |
| Section Title | 18px | 600 | Tiêu đề section, card header |
| Metric Value | 28–36px | 700 | Số KPI lớn |
| Body | 14px | 400 | Nội dung thông thường |
| Label / Caption | 12px | 500 | Label biểu đồ, ghi chú |
| Table Cell | 13px | 400 | Dữ liệu trong bảng |

---

## 3. Layout & Spacing

- **Grid**: 12 cột, gutter 16px
- **Container max-width**: 1280px, centered
- **Card padding**: 20px
- **Section gap**: 24px
- **Border radius**: 8px (card), 4px (button, badge), 12px (metric card lớn)

### Dashboard Structure

```
┌─────────────────────────────────────────┐
│  Header (logo + title + last updated)   │
├─────────────────────────────────────────┤
│  KPI Row: [Card] [Card] [Card] [Card]   │
├────────────────────┬────────────────────┤
│  Chart chính       │  Chart phụ / bảng  │
├────────────────────┴────────────────────┤
│  Bảng chi tiết (full width)             │
└─────────────────────────────────────────┘
```

---

## 4. Components

### KPI Card

```
┌──────────────────────┐
│ LABEL          [icon]│
│                      │
│  1.234.567           │  ← font 32px bold, color: --color-text
│  ▲ +12.5%            │  ← green nếu tăng, red nếu giảm
└──────────────────────┘
```

- Background: `--color-surface`
- Border: 1px solid `--color-border`
- Border-top: 3px solid `--color-gold` (accent)
- Shadow: `0 2px 8px rgba(0,0,0,0.06)`

### Chart

- Library: **Chart.js** hoặc **ECharts**
- Màu series chính: `#F5A623`, `#1565C0`, `#00A651`, `#E53935`
- Grid line: `rgba(0,0,0,0.06)`
- Tooltip background: `#1A1D2E` (dark), text white
- Không dùng 3D chart

### Bảng dữ liệu

- Header row: background `--color-surface-dark`, text white, font 12px 600 uppercase
- Alternating row: row chẵn background `#F8F9FA`
- Hover row: background `#FFF8E7` (gold tint nhẹ)
- Số dương: `--color-green`
- Số âm: `--color-danger`
- Align: text left, số right

### Badge / Status

| Status | Color |
|--------|-------|
| Active / Tốt | green `#00A651` |
| Pending | gold `#F5A623` |
| Inactive / Xấu | red `#E53935` |
| Neutral | gray `#6C757D` |

### Header

- Background: `--color-surface-dark` (`#1A1D2E`)
- Logo HSB bên trái
- Tiêu đề dashboard center hoặc kế logo
- "Last updated: DD/MM/YYYY HH:mm WIB" góc phải — font 12px, màu `rgba(255,255,255,0.6)`

---

## 5. Iconography

- Dùng **Lucide Icons** hoặc **Heroicons** (SVG inline, nhẹ, không cần CDN lớn)
- Size chuẩn: 16px (inline), 20px (card icon), 24px (nav)
- Màu icon theo context: gold cho KPI, green/red cho trend

---

## 6. Số & Định dạng

- Số tiền: `Rp 1.234.567` (dấu `.` phân cách nghìn, locale id-ID)
- Phần trăm: `+12,5%` (dấu `,` thập phân)
- Ngày: `22 Mei 2026` hoặc `22/05/2026`
- Số lớn (> 1 triệu): có thể rút gọn `1,2 Jt` hoặc `1.2M`

---

## 7. Nguyên tắc chung

1. **Data first** — Không trang trí thừa, mọi element phải có mục đích
2. **Consistent** — Dùng token màu, không hardcode hex trong JS/CSS
3. **Readable** — Contrast tối thiểu 4.5:1 (WCAG AA)
4. **Responsive** — Mobile tối thiểu 375px, desktop 1280px
5. **Fast** — Không dùng thư viện nặng, ưu tiên vanilla + 1 chart lib
6. **HSB Brand** — Gold accent là điểm nhấn chính, không lạm dụng màu sắc
