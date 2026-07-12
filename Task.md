# Eksportir — Project Task Board

> **Status ringkas (2026-07-11):** DB schema ✅ · Auth+RolesGuard ✅ · Category/Store/Product API ✅ · Order API (e2e 30/30) ✅ · Storage R2+Upload ✅ · Seller product UI ✅ · Landing katalog + detail produk ✅ · Buyer Dashboard (order flow + chat) ✅ · Seller Dashboard order UI + SidebarShell migration ✅ · Shared `OrderStatusBadge`/`OrderStatusTimeline` di buyer & seller ✅ (PLAN.md 100% selesai)
> UI hidup: `/` (katalog), `/products/[id]` (+ add to cart), `/seller/{login,register,store,products,orders,orders/[id]}`, `/buyer/{login,register,dashboard,orders,orders/[id],orders/new}`, `/admin/{login,categories,stores}`
> API: `http://localhost:3001/api` · Docs (Scalar): `http://localhost:3001/docs` · FE: `http://localhost:3000`
> Data fetching FE: **SWR + Axios**. Upload: **upload-first + staging→commit**. Seeder: `npm run seed:sellers` (4 seller, pw `seller123`). Berikutnya: **[PRIORITAS] Sinkronisasi field buyer detail page ↔ backend** · Admin order UI · Exchange Rate · SellerProfile bank info.

## Ringkasan Progres per Fase

> ✅ selesai · 🔄 sebagian · 🔲 belum mulai. Detail centang ada di tiap fase di bawah.

| Backend                   | Status                                           |     | Frontend                  | Status                                           |
| ------------------------- | ------------------------------------------------ | --- | ------------------------- | ------------------------------------------------ |
| 0 · Setup & Infra         | ✅                                               |     | 0 · Setup & UI Components | 🔄 (auth state belum)                            |
| 1 · DB Schema             | ✅                                               |     | 0B · i18n (next-intl)     | 🔲                                               |
| 2 · Auth                  | 🔄 (refresh/logout/change-pw/create-admin belum) |     | 0C · Currency display     | 🔲                                               |
| 3 · User & Profile        | 🔲                                               |     | 1 · Auth Pages            | 🔄 (middleware & refresh belum)                  |
| 4 · Store                 | ✅                                               |     | 2 · Public Pages          | 🔄 (katalog+detail ✅, toko belum)               |
| 5 · Product (+Category)   | ✅                                               |     | 3 · Buyer Dashboard       | 🔄 (order flow+chat ✅, txHash+dokumen belum)    |
| 6 · Order                 | ✅ (backend)                                     |     | 4 · Seller Dashboard      | 🔄 (produk+toko+landing+order UI ✅, bank belum) |
| 7 · Transaction/Escrow    | 🔲                                               |     | 5 · Admin Dashboard       | 🔄 (kategori+verifikasi toko ✅, sisanya belum)  |
| 8 · QC                    | 🔲                                               |     | 6 · Super Admin Dashboard | 🔲                                               |
| 9 · Export Document       | 🔲                                               |     | 7 · Notification & UX     | 🔲                                               |
| 10A · Exchange Rate       | 🔲 (model ✅)                                    |     |                           |                                                  |
| 10B · Storage R2 + Upload | ✅                                               |     |                           |                                                  |
| 11 · Notification         | 🔲                                               |     |                           |                                                  |
| 12 · Admin                | 🔲                                               |     |                           |                                                  |

## Project Context

Platform B2B ekspor barang dari Indonesia (fokus Bali) ke pasar internasional (Rusia & CIS).
Platform berperan sebagai **middleman** antara Seller dan Buyer dengan proses:

- Negosiasi difasilitasi admin (ruang chat 3 pihak)
- QC (Quality Check) fisik oleh tim platform
- Escrow dana dalam USDT hingga pengiriman selesai
- Dokumentasi ekspor digital
- Koordinasi logistik ke bandara/pelabuhan

**Target user:** Seller (UMKM Bali), Buyer (importir Rusia/CIS), Admin, Super Admin
**Traffic:** Rendah (B2B), tapi transaksi bernilai & volume besar

---

## Tech Stack

| Layer        | Teknologi                                                          |
| ------------ | ------------------------------------------------------------------ |
| Frontend     | Next.js 16, Tailwind CSS v4, TypeScript                            |
| Backend      | NestJS 11, TypeScript, ESM (`"type": "module"`)                    |
| Database     | **Supabase** (PostgreSQL managed) + Prisma ORM v7                  |
| Auth         | JWT (Access Token) — 3 endpoint terpisah per user type             |
| File Storage | Cloudflare R2 (S3-compatible)                                      |
| Validation   | class-validator + class-transformer                                |
| API Client   | Axios + SWR                                                        |
| i18n         | next-intl (EN + RU untuk public/buyer, EN + ID untuk seller/admin) |

---

## Infra & Environment Notes

### Database — Supabase (sejak 2026-07-12)

- DB lokal (Docker PostgreSQL) **sudah dihapus**. Satu-satunya DB adalah Supabase.
- `DATABASE_URL` → pooler port **6543** (`?pgbouncer=true`) — dipakai runtime app (PrismaService).
- `DIRECT_URL` → pooler port **5432** — dipakai Prisma CLI (`prisma migrate deploy`, `prisma db seed`).
- `docker-compose.yml` di VPS tidak lagi punya service `postgres` — hanya `api`, `nginx`, `umkm-api`.
- Local dev: cukup `npm run start:dev` di `eksportir-api/`, tidak perlu Docker untuk DB.

### ⚠️ Prisma Migrate — Wajib via Jaringan TELKOMSEL

- `prisma migrate deploy` / `prisma migrate dev` / `prisma generate` ke Supabase **hanya berhasil dari koneksi Telkomsel**.
- Koneksi lain (WiFi tertentu, ISP lain) konsisten gagal — diduga ada pemblokiran port atau routing ke server Supabase AP-Southeast.
- **Solusi:** pastikan laptop terhubung ke hotspot Telkomsel saat menjalankan perintah Prisma migrate.

---

## Folder Structure

```
eksportir/
├── Task.md                  ← File ini
├── CLAUDE.md                ← Briefing untuk AI (arsitektur, decisions)
├── README.md                ← Dokumentasi (English)
├── README_IND.md            ← Dokumentasi (Indonesia)
├── next-tailwind-ui/        ← Frontend Next.js
└── eksportir-api/           ← Backend NestJS
```

---

## Role System

| Role          | Deskripsi                                     |
| ------------- | --------------------------------------------- |
| `SELLER`      | Daftar toko, kelola produk, lihat order masuk |
| `BUYER`       | Browse produk, ajukan order, pantau status    |
| `ADMIN`       | Proses transaksi, QC, kelola dokumen          |
| `SUPER_ADMIN` | Full akses, manajemen user, laporan           |

> ⚠️ Buyer dan Seller adalah **table terpisah** (bukan 1 table User). Lihat CLAUDE.md.

---

## Transaction Flow (Core Business Logic)

```
Seller listing produk (harga IDR)
      │
BUYER ajukan order request
      │
Ruang chat terbuka: Buyer + Seller + Admin (negosiasi)
      │
Admin "lock" deal → harga final dikunci dalam USD
      │
BUYER transfer USDT → escrow platform
      │
ADMIN assign QC → tim QC lakukan inspeksi fisik
      │
QC lolos → ADMIN siapkan dokumen ekspor
      │
Barang dikirim ke bandara/pelabuhan
      │
Pengiriman dikonfirmasi → platform konversi USDT → IDR → transfer ke SELLER
      │
Transaksi selesai
```

---

---

# BACKEND TASKS (`eksportir-api/`)

## ✅ PHASE 0 — Setup & Infrastructure

- [x] NestJS v11 scaffolding
- [x] Install core dependencies (`@nestjs/config`, `@nestjs/jwt`, `passport`, `prisma`, `bcrypt`, `class-validator`, dll)
- [x] Prisma init (`prisma/schema.prisma`, `prisma.config.ts`)
- [x] Setup `.env` dan `ConfigModule` global
- [x] Setup `ValidationPipe` global di `main.ts`
- [x] Setup Prisma Service (`src/prisma/prisma.service.ts` — injectable, global)
- [x] Koneksi ke PostgreSQL via Docker (port 5432)
- [x] Struktur folder module awal (`src/prisma/`, `src/auth/`)
- [x] Docker Compose setup (PostgreSQL 17 Alpine)
- [x] Seed SuperAdmin default (`superadmin@eksportir.com` / `superadmin123`)
- [x] CORS enabled (`http://localhost:3000` + `http://localhost:3002`)
- [x] Scalar API docs di `http://localhost:3001/docs`

---

## ✅ PHASE 1 — Database Schema (Prisma)

- [x] Model `Buyer` (id, email, password, phone, nama, avatar, isActive, timestamps)
- [x] Model `Seller` (id, email, password, phone, nama, avatar, isActive, isVerified, timestamps)
- [x] Model `AdminUser` (id, email, password, nama, role: ADMIN|SUPER_ADMIN, isActive, timestamps)
- [x] Jalankan `prisma migrate dev` pertama (`20260601_init`)
- [x] Model `Store` (milik Seller — nama toko, deskripsi, status verifikasi)
- [x] Model `Category` (master table, dikelola Admin/SuperAdmin)
- [x] Model `Product` (nama, deskripsi, categoryId, storeId, isActive)
- [x] Model `ProductImage` (gallery foto produk, bisa banyak)
- [x] Model `ProductVariant` (nama free-text, harga IDR, stok, foto nullable, sku opsional)
- [x] Model `SellerProfile` (info rekening bank: bankName, bankAccountNumber, bankAccountName)
- [x] Model `Order` (buyer, seller, status, harga final USD, timestamps)
- [x] Model `OrderItem` (detail item per order — snapshot nama/harga produk)
- [x] Model `OrderMessage` (chat 3 pihak: buyer + seller + admin per order)
- [x] Model `Transaction` (USDT escrow — lihat CLAUDE.md untuk field detail)
- [x] Model `QCReport` (hasil inspeksi, foto, catatan, status)
- [x] Model `ExportDocument` (jenis dokumen, file URL, status)
- [x] Model `Notification` (user, pesan, read status)
- [x] Migrasi `add_store_category_product` + `add_order_transaction_qc_docs_notif_rate`

---

## 🔄 PHASE 2 — Auth Module

- [x] `POST /auth/buyer/register` — registrasi Buyer baru
- [x] `POST /auth/buyer/login` — login Buyer
- [x] `POST /auth/seller/register` — registrasi Seller baru (isVerified: false)
- [x] `POST /auth/seller/login` — login Seller
- [x] `POST /auth/admin/login` — login Admin & SuperAdmin (dibedakan dari `role` di response)
- [x] JWT Strategy (single strategy, dispatch berdasarkan `userType` di payload)
- [x] `JwtAuthGuard` (protect routes)
- [x] Password hashing dengan `bcrypt`
- [ ] `POST /auth/refresh` — refresh access token
- [ ] `POST /auth/logout` — invalidate refresh token
- [ ] `PATCH /auth/admin/change-password` — Admin ganti password
- [x] `RolesGuard` + `@Roles()` decorator (+ `@CurrentUser()` decorator)
- [ ] SuperAdmin create Admin endpoint (`POST /admin/users`)

---

## 🔲 PHASE 3 — User & Profile Module

- [ ] `GET /users/me` — get profil sendiri (Buyer / Seller / Admin)
- [ ] `PATCH /users/me` — update profil
- [ ] `GET /users/:id` — (Admin) lihat profil user
- [ ] `GET /users` — (Admin) list semua user dengan filter/pagination

---

## ✅ PHASE 4 — Store Module (Seller)

- [x] `POST /stores` — Seller buat toko
- [x] `GET /stores/my` — Seller lihat tokonya
- [x] `PATCH /stores/my` — Seller update info toko
- [x] `GET /stores` — (Public) list toko aktif & terverifikasi
- [x] `GET /stores/:id` — (Public) detail toko
- [x] `PATCH /stores/:id/verify` — (Admin) verifikasi toko
- [x] `GET /stores/admin/all` — (Admin) list semua toko untuk verifikasi

---

## ✅ PHASE 5 — Product Module

- [x] `POST /products` — (Seller) tambah produk + minimal 1 variant
- [x] `GET /products/my` — (Seller) lihat produk miliknya
- [x] `PATCH /products/:id` — (Seller) update produk (variants/imageUrls = replace, reconcile R2)
- [x] `DELETE /products/:id` — (Seller) hapus produk (+ hapus foto di R2)
- [x] `GET /products` — (Public) list produk dengan filter/pagination/search
- [x] `GET /products/:id` — (Public) detail produk + variants
- [x] Upload foto produk (ProductImage gallery) ke R2 — via upload-first + `commitFile`
- [x] Upload foto variant (nullable, 1 per variant) ke R2 — via `commitFile` folder `variants/`
- [x] ✔ Terverifikasi e2e: upload→commit (tmp/→products/), ownership guard, filter publik, cleanup
- [x] **Frontend** Seller UI kelola produk (list/tambah/edit/hapus + FileUpload) — ✔ terverifikasi di browser
- [x] **Frontend** Public catalog (`/`) + detail produk (`/products/[id]`) — ✔ terverifikasi di browser
- [x] `GET /categories` — list kategori (public)
- [x] `POST /categories` — (Admin) buat kategori
- [x] `DELETE /categories/:id` — (Admin) hapus kategori (ditolak jika masih dipakai produk)

---

## ✅ PHASE 6 — Order Module (backend e2e ✅ 30/30)

- [x] `POST /orders` — (Buyer) buat order request: `{ sellerId, items[], note? }`, 1 order = 1 seller, item validasi + snapshot nama/harga, `note` jadi pesan pembuka chat
- [x] `GET /orders/my` — role-aware (Buyer: order saya, Seller: order masuk ke toko)
- [x] `GET /orders/:id` — detail + items + seluruh chat (cek akses: buyer pemilik / seller toko / admin)
- [x] `PATCH /orders/:id/status` — (Admin) majukan status fulfillment (blok status terminal)
- [x] `PATCH /orders/:id/lock` — (Admin) kunci harga final USD → `CONFIRMED` (+ `lockedAt`)
- [x] Order status flow: `PENDING` → `NEGOTIATION` (auto saat ada pesan) → `CONFIRMED` (lock) → `PAID` → `QC_PROCESS` → `SHIPPING` → `DELIVERED` → `COMPLETED`
- [x] `PATCH /orders/:id/cancel` — (Buyer) cancel, hanya saat `PENDING`/`NEGOTIATION`
- [x] `POST /orders/:id/messages` — kirim pesan ke ruang negosiasi 3 pihak
- [x] `GET /orders` — (Admin) list semua order + filter `?status=`
- [x] List pesan negosiasi → sudah embedded di `GET /orders/:id` (1 fetch), tak perlu endpoint terpisah
- [x] Migrasi `add_order_note` (kolom `note` di `orders`)
- Catatan: Cart hidup di FE (localStorage), checkout per-seller. Backend tak butuh tabel Cart.

---

## 🔲 PHASE 7 — Transaction / Escrow Module

- [ ] `POST /transactions/:orderId/deposit` — (Buyer) submit txHash USDT
- [ ] `PATCH /transactions/:id/verify` — (Admin) verifikasi txHash di blockchain
- [ ] `PATCH /transactions/:id/release` — (Admin) cairkan dana ke Seller (konversi USDT → IDR)
- [ ] History transaksi per order
- [ ] Simpan: amountUsdt, amountUsd, usdtToIdrRate, amountIdr, walletAddress, txHash, network, disbursedAt

---

## 🔲 PHASE 8 — QC Module

- [ ] `POST /qc/:orderId` — (Admin) buat laporan QC
- [ ] Upload foto hasil QC ke R2
- [ ] `PATCH /qc/:id` — update hasil QC
- [ ] `GET /qc/:orderId` — lihat laporan QC per order
- [ ] QC status: `PENDING` → `IN_PROGRESS` → `PASSED` → `FAILED`

---

## 🔲 PHASE 9 — Export Document Module

- [ ] `POST /documents/:orderId` — (Admin) upload dokumen ekspor
- [ ] Tipe dokumen: Invoice, Packing List, COO, Phytosanitary, dll
- [ ] `GET /documents/:orderId` — list dokumen per order
- [ ] `DELETE /documents/:id` — hapus dokumen

---

## 🔲 PHASE 10A — Exchange Rate Module (Backend)

> Kurs disimpan di DB, diperbarui via cron. Admin bisa pantau, SuperAdmin bisa override manual.
> Pasangan: USD→RUB, USD→CNY, USDT→IDR.

- [x] Model `ExchangeRate` di Prisma (fromCurrency, toCurrency, rate, source: AUTO|MANUAL, updatedBy, updatedAt)
- [ ] Migrasi & seed data kurs awal (input manual pertama kali)
- [ ] `ExchangeRateService` — get kurs terkini per pasangan
- [ ] `GET /exchange-rates` — (Public) ambil semua kurs terkini
- [ ] `GET /exchange-rates/:from/:to` — (Public) ambil kurs spesifik
- [ ] `PATCH /exchange-rates/:from/:to` — (SuperAdmin only) input/update kurs manual
- [ ] Catat `source: MANUAL` dan `updatedBy` saat update
- [ ] `GET /admin/exchange-rates/history` — (Admin) riwayat perubahan kurs
- [ ] _(Future)_ Cron job otomatis dari frankfurter.app (USD→RUB, USD→CNY) + CoinGecko (USDT→IDR)

---

## ✅ PHASE 10B — Storage Module (Cloudflare R2) + Upload

- [x] Setup `StorageService` dengan AWS SDK v3 (`src/storage/`, `@Global`)
- [x] `uploadFile(file, folder)` — upload ke R2, return URL publik
- [x] `deleteFile(publicFileUrl)` — hapus dari R2
- [x] Isi kredensial R2 di `.env` (bucket `eksportir-prod`, public URL aktif)
- [x] Pola **upload-first + staging→commit**: `uploadTemp()`, `commitFile(tmpUrl, dest)`, `cleanupTemp()`
- [x] `POST /uploads` — endpoint universal (1 file → `tmp/`), validasi size 2MB + MIME + magic bytes
- [x] Cron harian 03:00 (`@nestjs/schedule`) — hapus `tmp/` > 24 jam (orphan cleanup)
- [ ] `getSignedUrl(key)` — presigned URL (opsional; saat ini pakai public bucket URL)

---

## 🔲 PHASE 11 — Notification Module

- [ ] `GET /notifications` — list notifikasi user
- [ ] `PATCH /notifications/:id/read` — tandai dibaca
- [ ] `PATCH /notifications/read-all` — tandai semua dibaca
- [ ] Internal `NotificationService` untuk trigger dari module lain

---

## 🔲 PHASE 12 — Admin Module

- [ ] Dashboard summary (total order, revenue, pending QC, dll)
- [ ] `GET /admin/orders` — list semua order dengan filter
- [ ] `GET /admin/users` — manajemen user (Buyer + Seller)
- [ ] `PATCH /admin/users/:id/status` — aktif/nonaktif user
- [ ] `GET /admin/transactions` — semua transaksi

---

---

# FRONTEND TASKS (`next-tailwind-ui/`)

## 🔄 PHASE 0 — Setup & UI Components

- [x] Next.js 16 scaffolding
- [x] Tailwind CSS v4 setup
- [x] UI Components library (Button, Input, Modal, Table, Badge, Card, Alert, FileUpload, Pagination, dll — Storybook)
- [x] Setup `axios` wrapper (`lib/axios.ts`) dengan request & response interceptors
- [x] Install SWR untuk async data fetching
- [x] Setup `.env.local` dengan `NEXT_PUBLIC_API_URL=http://localhost:3001`
- [x] `SidebarShell` — layout wrapper sidebar kiri (desktop) + bottom navbar (mobile) untuk semua dashboard
- [x] `BottomNav` — mobile-only fixed bottom navigation
- [x] Shared `OrderStatusBadge` + `OrderStatusTimeline` (`app/_components/`) — dipakai buyer & seller
- [x] Nav config seller (`app/seller/_nav.ts`) — Dashboard · Produk · Toko · Pesanan
- [ ] Setup global auth state (Zustand / Context)
- [ ] Layout dasar Admin (pakai SidebarShell)

---

## 🔲 PHASE 0B — Internationalization (i18n) Setup

> next-intl untuk locale routing. UI elements wajib diterjemahkan, konten produk/chat TIDAK.

### Bahasa per area

| Area             | Bahasa tersedia | Default |
| ---------------- | --------------- | ------- |
| Public + Buyer   | EN, RU          | EN      |
| Seller dashboard | EN, ID          | EN      |
| Admin dashboard  | EN, ID          | EN      |

### Tasks

- [ ] Install & konfigurasi `next-intl`
- [ ] Setup locale routing: `/en/...` dan `/ru/...` untuk public/buyer
- [ ] Buat file translation: `messages/en.json`, `messages/ru.json`, `messages/id.json`
- [ ] `middleware.ts` — detect locale dari Accept-Language header, cookie, atau URL prefix
- [ ] Komponen `LocaleSwitcher` — tombol ganti bahasa (EN ↔ RU atau EN ↔ ID)
- [ ] Terjemahkan semua UI component yang sudah ada (Button label, placeholder, alert text)
- [ ] Integrasi `LocaleSwitcher` ke Navbar public, Navbar seller, Navbar admin

---

## 🔲 PHASE 0C — Currency Display Setup (Frontend)

> Default tampilan harga dalam USD. Buyer bisa switch ke RUB atau CNY (display only, bukan payment).

- [ ] Komponen `CurrencySelector` — dropdown USD / RUB / CNY
- [ ] Global currency state (Zustand atau Context) — persist di localStorage
- [ ] Hook `usePrice(amountUsd)` — konversi ke currency yang dipilih pakai kurs dari API
- [ ] Integrasi `CurrencySelector` di Navbar public (terlihat saat browsing produk)
- [ ] Tampilkan harga yang sudah dikonversi di card produk, halaman detail produk, summary order
- [ ] Disclaimer kecil di bawah harga: _"Prices in RUB/CNY are indicative. Actual payment in USDT."_
- [ ] Fetch kurs dari `GET /exchange-rates` via SWR (cache, auto-refresh interval)

---

## 🔄 PHASE 1 — Auth Pages

- [x] Halaman Login Admin (`/admin/login`) — pakai komponen Input + Button dari Storybook
- [x] Halaman Login Buyer (`/buyer/login`)
- [x] Halaman Login Seller (`/seller/login`)
- [x] Halaman Register Buyer (`/buyer/register`)
- [x] Halaman Register Seller (`/seller/register`) — form toko menyusul di Seller dashboard (Store dibuat via `POST /stores` setelah login, bukan saat register)
- [x] Komponen `AuthShell` (`app/_components/AuthShell.tsx`) — chrome bersama untuk semua halaman auth
- [ ] Protected route middleware (`middleware.ts`)
- [ ] Auto-refresh token

---

## 🔄 PHASE 2 — Public Pages

- [x] Homepage / katalog (`/`) — navbar + hero + search + filter kategori + grid + pagination (IDR). ✔ verified
- [x] List produk (filter, search, pagination) — menyatu di homepage
- [x] Halaman detail produk (`/products/[id]`) — galeri + variant + harga. ✔ verified
- [ ] Halaman list toko
- [ ] Halaman detail toko
- [x] Seeder: 4 seller × 5 produk (`npm run seed:sellers`, password `seller123`)
  > Harga tampil IDR. Currency display USD/RUB/CNY = Phase 0C (belum).

---

## � PHASE 3 — Buyer Dashboard

> ⚠️ **Prioritas berikutnya:** halaman buyer detail masih memakai nama field yang tidak sinkron dengan response backend (`content`, `nama`, `harga`, `qty`, `senderType` uppercase, `senderName`). Backend sebenarnya mengembalikan `pesan`, `namaProduk`, `namaVariant`, `hargaIdr`, `quantity`, `senderType` lowercase (`buyer`/`seller`/`admin`), dan tidak ada `senderName`. Halaman seller sudah benar; buyer perlu diselaraskan agar chat & item pesanan render dengan benar.

- [ ] **[PRIORITAS]** Sinkronkan field FE ↔ BE di `buyer/orders/[id]/page.tsx`: `content`→`pesan`, `nama`→`namaProduk`, `harga`→`hargaIdr`, `qty`→`quantity`, `senderType` uppercase→lowercase, hapus dependency `senderName` (pakai label statis "Buyer"/"Seller"/"Admin" sesuai `senderType`)
- [ ] **[PRIORITAS]** Cek juga `buyer/orders/page.tsx` & `buyer/orders/new/page.tsx` untuk field yang sama
- [x] Dashboard Buyer (`/buyer`) — summary card total/aktif/selesai + recent orders + quick links
- [x] Halaman list order saya (`/buyer/orders`) — list semua order dengan status badge
- [x] Halaman detail order (`/buyer/orders/[id]`) — timeline status + items + chat negosiasi 3 pihak + cancel
- [x] Halaman buat order baru (`/buyer/orders/new`) — checkout dari cart localStorage + note + qty stepper
- [x] Add to Cart di halaman detail produk — pilih variant + qty + toast + warning ganti seller
- [ ] Submit txHash USDT (bukti bayar) — tunggu Phase 7 Backend
- [ ] Lihat dokumen ekspor — tunggu Phase 9 Backend

---

## 🔄 PHASE 4 — Seller Dashboard

- [x] Dashboard Seller (`/seller`) — pakai SidebarShell + summary card (total/perlu dibalas/selesai) + recent orders + quick links
- [x] Halaman kelola produk (`/seller/products` list, `/new`, `/[id]/edit`) — dimigrasi ke SidebarShell
- [x] Kelola variant produk (minimal 1, tambah/hapus baris di form)
- [x] Upload foto produk (gallery + per variant) via komponen `FileUpload`
- [x] Halaman order masuk (`/seller/orders`) — list + filter tab (Semua/Perlu Dibalas/Diproses/Selesai·Batal) dengan counter
- [x] Halaman detail order (`/seller/orders/[id]`) — timeline status + items + ruang chat 3 pihak (polling 10 dtk)
- [x] Halaman kelola toko (`/seller/store`) — dimigrasi ke SidebarShell + status verifikasi
- [ ] Info rekening bank (SellerProfile) di halaman toko

---

## 🔲 PHASE 5 — Admin Dashboard

- [ ] Dashboard Admin (statistik platform)
- [ ] Halaman kelola semua order + ruang chat negosiasi
- [ ] Halaman proses QC (input hasil, upload foto)
- [ ] Halaman kelola dokumen ekspor (upload, list per order)
- [ ] Halaman verifikasi pembayaran (txHash blockchain)
- [x] Halaman verifikasi toko Seller (`/admin/stores`)
- [x] Halaman kelola kategori (`/admin/categories`)

---

## 🔲 PHASE 6 — Super Admin Dashboard

- [ ] Semua fitur Admin +
- [ ] Halaman manajemen user (list Buyer + Seller + Admin, aktif/nonaktif)
- [ ] Halaman buat akun Admin baru (+ set password)
- [ ] Halaman laporan & analitik (revenue, volume, order)

---

## 🔲 PHASE 7 — Notification & UX

- [ ] Komponen notifikasi (dropdown di navbar)
- [ ] Toast notification untuk aksi (sukses, error)
- [ ] Loading skeleton untuk tabel & list
- [ ] Empty state komponen
- [ ] Error boundary dan halaman 404/500

---

---

# DEPLOYMENT (Later)

- [ ] Setup environment production (Railway / Render / VPS)
- [ ] CI/CD pipeline (GitHub Actions)
- [ ] Setup domain & SSL
- [ ] PostgreSQL production database
- [ ] Cloudflare R2 bucket production
- [ ] Environment variables production

---

## Progress Legend

| Simbol | Status                  |
| ------ | ----------------------- |
| `[x]`  | Selesai ✅              |
| `[ ]`  | Belum dikerjakan 🔲     |
| `[~]`  | Sedang dikerjakan 🔄    |
| `[-]`  | Di-skip / tidak jadi ❌ |
