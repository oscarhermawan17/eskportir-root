# Eksportir — Project Task Board

## Project Context

Platform B2B ekspor barang dari Indonesia (fokus Bali) ke pasar internasional (Rusia & CIS).
Platform berperan sebagai **middleman** antara Seller dan Buyer dengan proses:
- Negosiasi difasilitasi admin
- QC (Quality Check) fisik oleh tim platform
- Escrow dana hingga pengiriman selesai
- Dokumentasi ekspor digital
- Koordinasi logistik ke bandara/pelabuhan

**Target user:** Seller (UMKM Bali), Buyer (importir Rusia/CIS), Admin, Super Admin
**Traffic:** Rendah (B2B), tapi transaksi bernilai & volume besar

---

## Tech Stack

| Layer | Teknologi |
|-------|-----------|
| Frontend | Next.js 15, Tailwind CSS, TypeScript |
| Backend | NestJS 11, TypeScript |
| Database | PostgreSQL + Prisma ORM |
| Auth | JWT (Access Token + Refresh Token) |
| File Storage | Cloudflare R2 (S3-compatible) |
| Validation | class-validator + class-transformer |

---

## Folder Structure

```
eksportir/
├── Task.md                  ← File ini
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

---

## Transaction Flow (Core Business Logic)

```
BUYER ajukan order
      │
ADMIN review & fasilitasi negosiasi
      │
BUYER setuju → deposit dana (escrow)
      │
ADMIN assign QC → tim QC lakukan inspeksi fisik
      │
QC lolos → ADMIN siapkan dokumen ekspor
      │
Barang dikirim ke bandara/pelabuhan
      │
Pengiriman dikonfirmasi → dana dicairkan ke SELLER
      │
Transaksi selesai
```

---

---

# BACKEND TASKS (`eksportir-api/`)

## ✅ PHASE 0 — Setup & Infrastructure
- [x] NestJS v11 scaffolding
- [x] Install core dependencies (`@nestjs/config`, `@nestjs/jwt`, `passport`, `prisma`, `bcrypt`, `class-validator`, dll)
- [x] Prisma init (`prisma/schema.prisma`, `.env`)
- [ ] Setup `.env` dan `ConfigModule` global
- [ ] Setup `ValidationPipe` global di `main.ts`
- [ ] Setup Prisma Service (injectable `PrismaService`)
- [ ] Koneksi ke PostgreSQL (lokal / cloud)
- [ ] Struktur folder module awal

---

## 🔲 PHASE 1 — Database Schema (Prisma)
- [ ] Model `User` (id, email, password, role, profile, timestamps)
- [ ] Model `Store` (milik Seller — nama toko, deskripsi, status verifikasi)
- [ ] Model `Product` (nama, deskripsi, harga, stok, kategori, foto, store)
- [ ] Model `Order` (buyer, seller, produk, status, total, timestamps)
- [ ] Model `OrderItem` (detail item per order)
- [ ] Model `Transaction` (escrow status, jumlah dana, bukti transfer)
- [ ] Model `QCReport` (hasil inspeksi, foto, catatan, status)
- [ ] Model `ExportDocument` (jenis dokumen, file URL, status)
- [ ] Model `Notification` (user, pesan, read status)
- [ ] Jalankan `prisma migrate dev` pertama

---

## 🔲 PHASE 2 — Auth Module
- [ ] `POST /auth/register` — registrasi user (default role: BUYER)
- [ ] `POST /auth/login` — login, return access + refresh token
- [ ] `POST /auth/refresh` — refresh access token
- [ ] `POST /auth/logout` — invalidate refresh token
- [ ] JWT Strategy (`passport-jwt`)
- [ ] `JwtAuthGuard` (protect routes)
- [ ] `RolesGuard` + `@Roles()` decorator
- [ ] Password hashing dengan `bcrypt`

---

## 🔲 PHASE 3 — User & Profile Module
- [ ] `GET /users/me` — get profil sendiri
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
- [ ] `POST /products` — (Seller) tambah produk
- [ ] `GET /products/my` — (Seller) lihat produk miliknya
- [ ] `PATCH /products/:id` — (Seller) update produk
- [ ] `DELETE /products/:id` — (Seller) hapus produk
- [ ] `GET /products` — (Public) list produk dengan filter/pagination/search
- [ ] `GET /products/:id` — (Public) detail produk
- [ ] Upload foto produk ke Cloudflare R2

---

## 🔲 PHASE 6 — Order Module
- [ ] `POST /orders` — (Buyer) buat order
- [ ] `GET /orders/my` — (Buyer/Seller) lihat order sendiri
- [ ] `GET /orders/:id` — detail order
- [ ] `PATCH /orders/:id/status` — (Admin) update status order
- [ ] Order status flow: `PENDING` → `NEGOTIATION` → `CONFIRMED` → `PAID` → `QC_PROCESS` → `SHIPPING` → `DELIVERED` → `COMPLETED`
- [ ] `POST /orders/:id/cancel` — cancel order

---

## 🔲 PHASE 7 — Transaction / Escrow Module
- [ ] `POST /transactions/:orderId/deposit` — (Buyer) konfirmasi deposit
- [ ] Upload bukti transfer ke R2
- [ ] `PATCH /transactions/:id/verify` — (Admin) verifikasi pembayaran
- [ ] `PATCH /transactions/:id/release` — (Admin) cairkan dana ke Seller
- [ ] History transaksi per order

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

## 🔲 PHASE 10 — Storage Module (Cloudflare R2)
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
- [ ] `GET /admin/users` — manajemen user
- [ ] `PATCH /admin/users/:id/status` — aktif/nonaktif user
- [ ] `GET /admin/transactions` — semua transaksi

---

---

# FRONTEND TASKS (`next-tailwind-ui/`)

## ✅ PHASE 0 — Setup & UI Components
- [x] Next.js 15 scaffolding
- [x] Tailwind CSS setup
- [x] UI Components library (Button, Input, Modal, Table, Badge, dll — Storybook)
- [ ] Setup `axios` / `fetch` wrapper untuk API calls
- [ ] Setup global state (Zustand / React Context untuk auth)
- [ ] Setup `.env.local` dengan `NEXT_PUBLIC_API_URL`
- [ ] Layout dasar (Sidebar, Navbar, Footer)

---

## 🔲 PHASE 1 — Auth Pages
- [ ] Halaman Login
- [ ] Halaman Register (Buyer)
- [ ] Halaman Register Seller (+ form toko)
- [ ] Protected route middleware (`middleware.ts`)
- [ ] Simpan token di `httpOnly cookie` atau `localStorage`
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
- [ ] Halaman detail order (timeline status)
- [ ] Upload bukti transfer
- [ ] Lihat dokumen ekspor

---

## 🔲 PHASE 4 — Seller Dashboard
- [ ] Dashboard Seller (ringkasan produk & order)
- [ ] Halaman kelola produk (list, tambah, edit, hapus)
- [ ] Upload foto produk
- [ ] Halaman order masuk
- [ ] Halaman profil toko

---

## 🔲 PHASE 5 — Admin Dashboard
- [ ] Dashboard Admin (statistik platform)
- [ ] Halaman kelola semua order
- [ ] Halaman proses QC (input hasil, upload foto)
- [ ] Halaman kelola dokumen ekspor (upload, list per order)
- [ ] Halaman verifikasi pembayaran
- [ ] Halaman verifikasi toko Seller

---

## 🔲 PHASE 6 — Super Admin Dashboard
- [ ] Semua fitur Admin +
- [ ] Halaman manajemen user (list, aktif/nonaktif)
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
