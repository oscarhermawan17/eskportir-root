# PLAN — Seller Order UI + Migrasi SidebarShell

**Tanggal dibuat:** 2026-07-07
**Status:** Siap dikerjakan
**Scope:** Frontend Next.js (`next-tailwind-ui/`)

---

## Konteks

Backend Order sudah 100% selesai (30/30 e2e). Seller perlu UI untuk:

- Melihat order masuk dari Buyer
- Membalas chat negosiasi 3 pihak (Buyer + Seller + Admin)
- Memantau status order

Sekalian dalam task ini: migrasi semua halaman Seller yang masih pakai
`DashboardShell` lama (header sederhana) ke `SidebarShell` baru (sidebar kiri
desktop + bottom navbar mobile) agar semua halaman seller konsisten.

---

## Prinsip Pengerjaan

- Reuse `SidebarShell` dan `BottomNav` yang sudah dibuat di Buyer task
- Ekstrak `StatusBadge` dan `StatusTimeline` jadi shared component agar tidak
  duplikasi antara buyer dan seller pages
- Gunakan design tokens dari `globals.css` — tidak hardcode warna
- Import langsung dari file (bukan barrel) — Vercel best practice
- `"use client"` hanya pada komponen yang membutuhkan interaktivitas
- Data fetching via SWR + axios instance dari `@/lib/axios`

---

## File yang Akan Dibuat / Diedit

### Dibuat Baru (4 file)

```
next-tailwind-ui/
  app/
    _components/
      OrderStatusBadge.tsx        ← Badge status order (shared buyer & seller)
      OrderStatusTimeline.tsx     ← Timeline stepper (shared buyer & seller)
    seller/
      _nav.ts                     ← Nav config seller (shared semua halaman seller)
      orders/
        page.tsx                  ← List pesanan masuk + filter tab
        [id]/
          page.tsx                ← Detail order + chat negosiasi
```

### Diedit (7 file)

```
next-tailwind-ui/
  app/
    seller/
      page.tsx                         ← Pakai SidebarShell
      store/
        page.tsx                       ← Ganti DashboardShell → SidebarShell
      products/
        page.tsx                       ← Ganti DashboardShell → SidebarShell
        new/
          page.tsx                     ← Ganti DashboardShell → SidebarShell
        [id]/
          edit/
            page.tsx                   ← Ganti DashboardShell → SidebarShell
    buyer/
      orders/
        [id]/
          page.tsx                     ← Ganti StatusBadge & StatusTimeline lokal → shared import
        page.tsx                       ← Ganti StatusBadge lokal → shared import
```

---

## Step-by-Step Detail

### STEP 1 — Shared Components

#### `app/_components/OrderStatusBadge.tsx`

Props:

```ts
interface OrderStatusBadgeProps {
  status: string
  size?: "default" | "small"
}
```

Map status → variant Badge + label (Bahasa Indonesia):
| Status | Label | Badge Variant |
|--------|-------|---------------|
| PENDING | Pending | warning |
| NEGOTIATION | Negosiasi | secondary |
| CONFIRMED | Dikonfirmasi | primary |
| PAID | Dibayar | success |
| QC_PROCESS | QC | success |
| SHIPPING | Dikirim | success |
| DELIVERED | Terkirim | success |
| COMPLETED | Selesai | success |
| CANCELLED | Dibatalkan | danger |

Export: `export function OrderStatusBadge({ status, size = "small" }: ...)`

---

#### `app/_components/OrderStatusTimeline.tsx`

Props:

```ts
interface OrderStatusTimelineProps {
  currentStatus: string
}
```

Steps yang ditampilkan (urutan):
`PENDING → NEGOTIATION → CONFIRMED → PAID → QC_PROCESS → SHIPPING → DELIVERED → COMPLETED`

Jika status `CANCELLED`, tampilkan pesan "Order dibatalkan" dengan ikon cancel merah.

Visual:

- Node lingkaran: done = bg-primary + ikon check, active = bg-primary + ring, future = bg-surface-container-high
- Connector garis: done = bg-primary, future = bg-surface-container-high
- Label di bawah tiap node (text-[10px])
- Scrollable horizontal (`overflow-x-auto`) untuk tampilan mobile

---

### STEP 2 — Seller Nav Config

#### `app/seller/_nav.ts`

```ts
export const SELLER_BRAND = {
  initials: "EK",
  name: "Eksportir",
  subtitle: "Seller Portal",
}
```

**Nav Groups (Sidebar Desktop):**

```
[Section tanpa label]
  - Dashboard    icon: dashboard       href: /seller         id: seller-dashboard
  - Produk       icon: inventory_2     href: /seller/products  id: seller-products
  - Toko         icon: storefront      href: /seller/store    id: seller-store
  - Pesanan      icon: receipt_long    href: /seller/orders   id: seller-orders
```

**Footer Items (Sidebar, pinned bawah):**

```
  - Logout       icon: logout          id: seller-logout      (onClick: handleLogout)
```

**Bottom Nav (Mobile, max 4 item):**

```
  Dashboard  → /seller           id: seller-dashboard
  Produk     → /seller/products  id: seller-products
  Pesanan    → /seller/orders    id: seller-orders
  Toko       → /seller/store     id: seller-store
```

---

### STEP 3 — Migrasi Halaman Seller yang Ada

Perubahan di semua halaman ini **minimal** — hanya wrapper layout yang diganti.
Logic form, data fetching, dan UI konten **tidak disentuh sama sekali**.

Pola perubahan:

1. Hapus import `DashboardShell`
2. Tambah import `SidebarShell` dari `@/app/_components/SidebarShell`
3. Tambah import `SELLER_BRAND`, `SELLER_NAV_GROUPS`, `SELLER_FOOTER_ITEMS`, `SELLER_BOTTOM_NAV` dari `@/app/seller/_nav`
4. Tambah `handleLogout` function (sama di semua halaman)
5. Ganti `<DashboardShell icon="..." area="..." title="...">` dengan `<SidebarShell ...>`
6. Pindahkan judul halaman ke dalam konten (karena SidebarShell tidak punya prop `title`)
7. Set `activeItemId` sesuai halaman

**Mapping `activeItemId` per halaman:**
| Halaman | activeItemId |
|---------|-------------|
| `/seller` | `seller-dashboard` |
| `/seller/store` | `seller-store` |
| `/seller/products` | `seller-products` |
| `/seller/products/new` | `seller-products` |
| `/seller/products/[id]/edit` | `seller-products` |

---

### STEP 4 — `app/seller/orders/page.tsx`

**Data:** `GET /api/orders/my` via SWR (backend sudah role-aware, jika token = seller, return order masuk ke toko seller)

**Auth guard:** jika tidak ada token di localStorage → redirect `/seller/login`

**Filter Tab (client-side):**

```
[ Semua (N) ]  [ Perlu Dibalas (N) ]  [ Diproses (N) ]  [ Selesai/Batal (N) ]
```

Dimana N = jumlah order di tab tersebut (badge angka kecil di sebelah label).

Definisi filter:

- **Semua** → semua order
- **Perlu Dibalas** → status: `PENDING`, `NEGOTIATION`
- **Diproses** → status: `CONFIRMED`, `PAID`, `QC_PROCESS`, `SHIPPING`
- **Selesai/Batal** → status: `DELIVERED`, `COMPLETED`, `CANCELLED`

Filter berjalan murni di client dari data SWR yang sudah di-fetch — tidak ada request API tambahan.

**Layout tiap order card:**

```
┌────────────────────────────────────────────────┐
│ 👤 Nama Buyer                  [Status Badge]  │
│ 2 item: Kopi Arabika ×5, ...   [chevron_right] │
│ 5 Juli 2026                                    │
└────────────────────────────────────────────────┘
```

Klik card → `/seller/orders/[id]`

**Empty state:** Ikon receipt_long + teks "Belum ada pesanan masuk"

---

### STEP 5 — `app/seller/orders/[id]/page.tsx`

**Data:** `GET /api/orders/:id` via SWR (refresh interval 10 detik untuk pesan baru)
**Send message:** `POST /api/orders/:id/messages`

**Perbedaan vs buyer detail:**
| Aspek | Buyer | Seller |
|-------|-------|--------|
| Header | Tampilkan nama Seller | Tampilkan nama Buyer |
| Chat "isMine" | `senderType === "BUYER"` | `senderType === "SELLER"` |
| Tombol Cancel | Ada (jika PENDING/NEGOTIATION) | **Tidak ada** |
| Tombol aksi lain | — | — |

**Komponen yang dipakai (import dari shared):**

- `OrderStatusBadge` dari `@/app/_components/OrderStatusBadge`
- `OrderStatusTimeline` dari `@/app/_components/OrderStatusTimeline`
- `Button`, `Input` dari `@/stories/`
- `SidebarShell` dari `@/app/_components/SidebarShell`

**Layout halaman:**

```
← Kembali ke pesanan

Order #XXXXXXXX    [Status Badge]
Buyer: Nama Buyer · 5 Juli 2026

┌─ Status Pesanan ─────────────────────────────┐
│  [Timeline horizontal scrollable]            │
└──────────────────────────────────────────────┘

┌─ Item Pesanan ───────────────────────────────┐
│  Nama produk — variant        Rp xxx.xxx     │
│  Qty: N                                      │
├──────────────────────────────────────────────┤
│  [Harga final USD jika sudah dikunci Admin]  │
└──────────────────────────────────────────────┘

┌─ Ruang Negosiasi ────────────────────────────┐
│  Chat 3 pihak: Buyer · Seller · Admin        │
│                                              │
│  [Bubble pesan — scroll area]                │
│                                              │
├──────────────────────────────────────────────┤
│  [Input pesan]          [Kirim]              │
└──────────────────────────────────────────────┘
```

Form kirim pesan disembunyikan jika status `COMPLETED` atau `CANCELLED`.

---

### STEP 6 — Update Buyer Pages (reuse shared components)

Setelah `OrderStatusBadge` dan `OrderStatusTimeline` tersedia:

- `buyer/orders/page.tsx` → hapus `StatusBadge` lokal, import `OrderStatusBadge`
- `buyer/orders/[id]/page.tsx` → hapus `StatusBadge` + `StatusTimeline` lokal, import keduanya
- `buyer/page.tsx` → hapus `StatusBadge` lokal, import `OrderStatusBadge`

---

## Urutan Pengerjaan

```
1. OrderStatusBadge.tsx          (shared)
2. OrderStatusTimeline.tsx       (shared)
3. seller/_nav.ts
4. Migrasi seller/page.tsx
5. Migrasi seller/store/page.tsx
6. Migrasi seller/products/page.tsx
7. Migrasi seller/products/new/page.tsx
8. Migrasi seller/products/[id]/edit/page.tsx
9. seller/orders/page.tsx        (list + filter tab)
10. seller/orders/[id]/page.tsx  (detail + chat)
11. Update buyer pages (reuse shared components)
```

---

## Yang TIDAK Dikerjakan dalam Task Ini

- Seller tidak bisa cancel, lock harga, atau update status order (hak Admin/Buyer)
- Info rekening bank (SellerProfile) — task terpisah
- Protected route middleware (`middleware.ts`) — task terpisah
- i18n / Currency display — task terpisah

---

## Hasil Akhir yang Diharapkan

Setelah task ini selesai, flow berikut bisa berjalan penuh:

```
Buyer tambah ke cart → buat order → chat dengan Seller
         ↕
Seller lihat order masuk → balas chat → koordinasi dengan Admin
         ↕
Admin (belum ada UI) → lock harga → update status
```

Semua halaman Seller akan punya tampilan konsisten:
sidebar kiri (desktop) + bottom navbar (mobile).
