# Eksportir — Project Task Board

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

| Layer | Teknologi |
|-------|-----------|
| Frontend | Next.js 16, Tailwind CSS v4, TypeScript |
| Backend | NestJS 11, TypeScript, ESM (`"type": "module"`) |
| Database | PostgreSQL 17 (Docker) + Prisma ORM v7 |
| Auth | JWT (Access Token) — 3 endpoint terpisah per user type |
| File Storage | Cloudflare R2 (S3-compatible) |
| Validation | class-validator + class-transformer |
| API Client | Axios + SWR |
| i18n | next-intl (EN + RU untuk public/buyer, EN + ID untuk seller/admin) |

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

| Role | Deskripsi |
|------|-----------|
| `SELLER` | Daftar toko, kelola produk, lihat order masuk |
| `BUYER` | Browse produk, ajukan order, pantau status |
| `ADMIN` | Proses transaksi, QC, kelola dokumen |
| `SUPER_ADMIN` | Full akses, manajemen user, laporan |

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
- [x] CORS enabled (`http://localhost:3000`)
- [x] Scalar API docs di `http://localhost:3001/docs`

---

## 🔄 PHASE 1 — Database Schema (Prisma)
- [x] Model `Buyer` (id, email, password, phone, nama, avatar, isActive, timestamps)
- [x] Model `Seller` (id, email, password, phone, nama, avatar, isActive, isVerified, timestamps)
- [x] Model `AdminUser` (id, email, password, nama, role: ADMIN|SUPER_ADMIN, isActive, timestamps)
- [x] Jalankan `prisma migrate dev` pertama (`20260601_init`)
- [ ] Model `Store` (milik Seller — nama toko, deskripsi, status verifikasi)
- [ ] Model `Category` (master table, dikelola Admin/SuperAdmin)
- [ ] Model `Product` (nama, deskripsi, categoryId, storeId, isActive)
- [ ] Model `ProductImage` (gallery foto produk, bisa banyak)
- [ ] Model `ProductVariant` (nama free-text, harga IDR, stok, foto nullable, sku opsional)
- [ ] Model `Order` (buyer, seller, status, harga final USD, timestamps)
- [ ] Model `OrderItem` (detail item per order)
- [ ] Model `OrderMessage` (chat 3 pihak: buyer + seller + admin per order)
- [ ] Model `Transaction` (USDT escrow — lihat CLAUDE.md untuk field detail)
- [ ] Model `QCReport` (hasil inspeksi, foto, catatan, status)
- [ ] Model `ExportDocument` (jenis dokumen, file URL, status)
- [ ] Model `Notification` (user, pesan, read status)

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
- [ ] `RolesGuard` + `@Roles()` decorator
- [ ] SuperAdmin create Admin endpoint (`POST /admin/users`)

---

## 🔲 PHASE 3 — User & Profile Module
- [ ] `GET /users/me` — get profil sendiri (Buyer / Seller / Admin)
- [ ] `PATCH /users/me` — update profil
- [ ] `GET /users/:id` — (Admin) lihat profil user
- [ ] `GET /users` — (Admin) list semua user dengan filter/pagination

---

## 🔲 PHASE 4 — Store Module (Seller)
- [ ] `POST /stores` — Seller buat toko
- [ ] `GET /stores/my` — Seller lihat tokonya
- [ ] `PATCH /stores/my` — Seller update info toko
- [ ] `GET /stores` — (Public) list toko aktif
- [ ] `GET /stores/:id` — (Public) detail toko
- [ ] `PATCH /stores/:id/verify` — (Admin) verifikasi toko

---

## 🔲 PHASE 5 — Product Module
- [ ] `POST /products` — (Seller) tambah produk + minimal 1 variant
- [ ] `GET /products/my` — (Seller) lihat produk miliknya
- [ ] `PATCH /products/:id` — (Seller) update produk
- [ ] `DELETE /products/:id` — (Seller) hapus produk
- [ ] `GET /products` — (Public) list produk dengan filter/pagination/search
- [ ] `GET /products/:id` — (Public) detail produk + variants
- [ ] Upload foto produk (ProductImage gallery) ke Cloudflare R2
- [ ] Upload foto variant (nullable, 1 per variant) ke Cloudflare R2
- [ ] `GET /categories` — list kategori (public)
- [ ] `POST /categories` — (Admin) buat kategori
- [ ] `DELETE /categories/:id` — (Admin) hapus kategori

---

## 🔲 PHASE 6 — Order Module
- [ ] `POST /orders` — (Buyer) buat order request
- [ ] `GET /orders/my` — (Buyer/Seller) lihat order sendiri
- [ ] `GET /orders/:id` — detail order
- [ ] `PATCH /orders/:id/status` — (Admin) update status order
- [ ] Order status flow: `PENDING` → `NEGOTIATION` → `CONFIRMED` → `PAID` → `QC_PROCESS` → `SHIPPING` → `DELIVERED` → `COMPLETED`
- [ ] `POST /orders/:id/cancel` — cancel order
- [ ] `POST /orders/:id/messages` — kirim pesan di ruang negosiasi
- [ ] `GET /orders/:id/messages` — list pesan negosiasi

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

- [ ] Model `ExchangeRate` di Prisma (fromCurrency, toCurrency, rate, source: AUTO|MANUAL, updatedBy, updatedAt)
- [ ] Migrasi & seed data kurs awal (input manual pertama kali)
- [ ] `ExchangeRateService` — get kurs terkini per pasangan
- [ ] `GET /exchange-rates` — (Public) ambil semua kurs terkini
- [ ] `GET /exchange-rates/:from/:to` — (Public) ambil kurs spesifik
- [ ] `PATCH /exchange-rates/:from/:to` — (SuperAdmin only) input/update kurs manual
- [ ] Catat `source: MANUAL` dan `updatedBy` saat update
- [ ] `GET /admin/exchange-rates/history` — (Admin) riwayat perubahan kurs
- [ ] _(Future)_ Cron job otomatis dari frankfurter.app (USD→RUB, USD→CNY) + CoinGecko (USDT→IDR)

---

## 🔲 PHASE 10B — Storage Module (Cloudflare R2)
- [ ] Setup `StorageService` dengan AWS SDK v3
- [ ] `uploadFile(file, folder)` — upload ke R2
- [ ] `deleteFile(key)` — hapus dari R2
- [ ] `getSignedUrl(key)` — generate presigned URL

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

## ✅ PHASE 0 — Setup & UI Components
- [x] Next.js 16 scaffolding
- [x] Tailwind CSS v4 setup
- [x] UI Components library (Button, Input, Modal, Table, Badge, dll — Storybook)
- [x] Setup `axios` wrapper (`lib/axios.ts`) dengan request & response interceptors
- [x] Install SWR untuk async data fetching
- [x] Setup `.env.local` dengan `NEXT_PUBLIC_API_URL=http://localhost:3001`
- [ ] Setup global auth state (Zustand / Context)
- [ ] Layout dasar Admin (Sidebar, Navbar, Header)

---

## 🔲 PHASE 0B — Internationalization (i18n) Setup

> next-intl untuk locale routing. UI elements wajib diterjemahkan, konten produk/chat TIDAK.

### Bahasa per area
| Area | Bahasa tersedia | Default |
|------|----------------|---------|
| Public + Buyer | EN, RU | EN |
| Seller dashboard | EN, ID | EN |
| Admin dashboard | EN, ID | EN |

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
- [ ] Halaman Login Buyer
- [ ] Halaman Login Seller
- [ ] Halaman Register Buyer
- [ ] Halaman Register Seller (+ form toko)
- [ ] Protected route middleware (`middleware.ts`)
- [ ] Auto-refresh token

---

## 🔲 PHASE 2 — Public Pages
- [ ] Homepage (landing page, hero, featured products)
- [ ] Halaman list produk (filter, search, pagination)
- [ ] Halaman detail produk
- [ ] Halaman list toko
- [ ] Halaman detail toko

---

## 🔲 PHASE 3 — Buyer Dashboard
- [ ] Dashboard Buyer (ringkasan order)
- [ ] Halaman buat order baru
- [ ] Halaman list order saya
- [ ] Halaman detail order (timeline status + ruang chat negosiasi)
- [ ] Submit txHash USDT (bukti bayar)
- [ ] Lihat dokumen ekspor

---

## 🔲 PHASE 4 — Seller Dashboard
- [ ] Dashboard Seller (ringkasan produk & order)
- [ ] Halaman kelola produk (list, tambah, edit, hapus)
- [ ] Kelola variant produk (minimal 1)
- [ ] Upload foto produk (gallery + per variant)
- [ ] Halaman order masuk + ruang chat negosiasi
- [ ] Halaman profil toko + info rekening bank

---

## 🔲 PHASE 5 — Admin Dashboard
- [ ] Dashboard Admin (statistik platform)
- [ ] Halaman kelola semua order + ruang chat negosiasi
- [ ] Halaman proses QC (input hasil, upload foto)
- [ ] Halaman kelola dokumen ekspor (upload, list per order)
- [ ] Halaman verifikasi pembayaran (txHash blockchain)
- [ ] Halaman verifikasi toko Seller

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

| Simbol | Status |
|--------|--------|
| `[x]` | Selesai ✅ |
| `[ ]` | Belum dikerjakan 🔲 |
| `[~]` | Sedang dikerjakan 🔄 |
| `[-]` | Di-skip / tidak jadi ❌ |
