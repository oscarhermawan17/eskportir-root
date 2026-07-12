# CLAUDE.md — Eksportir Project Briefing

File ini dibaca otomatis oleh Claude Code di setiap sesi baru.
Berisi keputusan arsitektur, business rules, dan konteks yang tidak obvious dari kode.

---

## Project Overview

**Eksportir** — Platform B2B ekspor barang dari Indonesia (fokus Bali) ke pasar internasional (Rusia & CIS).
Platform berperan sebagai **middleman terkelola**: negosiasi, QC fisik, escrow dana, dokumen ekspor, koordinasi logistik.

Lihat `README.md` untuk overview lengkap dan `Task.md` untuk progress tracker.

---

## Status Saat Ini (snapshot per 2026-07-12)

### Sudah dikerjakan

- **DB schema lengkap** — semua model inti + enum sudah dimigrasi (Buyer, Seller, AdminUser, SellerProfile, Store, Category, Product, ProductImage, ProductVariant, Order, OrderItem, OrderMessage, Transaction, QCReport, ExportDocument, Notification, ExchangeRate).
- **Auth** — register/login Buyer, Seller, Admin (3 endpoint terpisah), JWT, `JwtAuthGuard`, `RolesGuard` + `@Roles()` + `@CurrentUser()`.
- **Category module** — `GET` (public), `POST`/`DELETE` (admin).
- **Store module** — `POST`/`GET my`/`PATCH my` (seller), `GET`/`GET :id` (public), `PATCH :id/verify` + `GET admin/all` (admin).
- **Product module** — `POST`/`GET my`/`PATCH :id`/`DELETE :id` (seller), `GET` (public, search+filter+pagination)/`GET :id` (public). Foto via pola upload-first + `commitFile`. Terverifikasi e2e.
- **Order module (Phase 6) — backend 100% (e2e 30/30).** Endpoint list, buat, detail, kirim pesan, cancel, admin lock harga USD, admin update status.
- **Upload module** — `POST /uploads` (semua user login): upload 1 gambar ke staging `tmp/` di R2, return `{ url }`. Validasi size 2MB + MIME + magic bytes.
- **Storage R2** — kredensial sudah terisi, bucket `eksportir-prod`, public URL aktif. Pola staging→commit lengkap (lihat "Keputusan Upload File").
- **Cron** — `@nestjs/schedule` aktif; cleanup `tmp/` tiap hari jam 03:00 (file > 24 jam dihapus).
- **Frontend Buyer & Seller order UI** — dashboard, list pesanan (filter tab), detail + chat 3 pihak (polling 10 dtk), cart FE + checkout per-seller.
- **SidebarShell + BottomNav** — semua halaman seller & buyer sudah pakai layout konsisten (sidebar desktop + bottom navbar mobile). Shared `OrderStatusBadge` + `OrderStatusTimeline` di `app/_components/`.

### Endpoint API yang hidup (`http://localhost:3001/api`)

`auth/{buyer,seller}/{register,login}`, `auth/admin/login`, `categories` (GET/POST/DELETE), `stores` (POST, GET, GET :id, GET my, PATCH my, PATCH :id/verify, GET admin/all), `products` (POST, GET my, PATCH :id, DELETE :id, GET, GET :id), `uploads` (POST), `orders` (POST [buyer], GET my [buyer/seller], GET :id [pemilik], POST :id/messages, PATCH :id/cancel [buyer], GET [admin], PATCH :id/lock [admin], PATCH :id/status [admin]).

> **Order module (Phase 6) — backend selesai, e2e 30/30.** 1 order = 1 seller; item di-snapshot (nama+harga IDR) saat dibuat; `note` buyer jadi pesan pembuka chat 3 pihak. Status: `PENDING` → `NEGOTIATION` (auto saat pesan pertama masuk) → `CONFIRMED` (admin `lock` harga final USD) → `PAID`/`QC_PROCESS`/`SHIPPING`/… (admin `status`). Cancel buyer hanya saat PENDING/NEGOTIATION. **Cart hidup di FE (localStorage), checkout per-seller — backend tak punya tabel Cart.** Detail order sudah meng-embed seluruh `messages` (1 fetch, tak perlu endpoint list pesan terpisah).

### Halaman UI yang sudah ada (`http://localhost:3000`)

| Path                                                                     | Untuk                                                                             |
| ------------------------------------------------------------------------ | --------------------------------------------------------------------------------- |
| `/`                                                                      | **Katalog publik** — grid produk, search, filter kategori, pagination (harga IDR) |
| `/products/[id]`                                                         | **Detail produk publik** — galeri + variant + harga                               |
| `/admin/login`                                                           | Login Admin/SuperAdmin                                                            |
| `/buyer/login`, `/buyer/register`                                        | Auth Buyer                                                                        |
| `/seller/login`, `/seller/register`                                      | Auth Seller                                                                       |
| `/seller`                                                                | Dashboard Seller (summary + recent orders + quick links)                          |
| `/seller/store`                                                          | Seller buat/edit toko                                                             |
| `/seller/products`, `/seller/products/new`, `/seller/products/[id]/edit` | Seller kelola produk                                                              |
| `/seller/orders`, `/seller/orders/[id]`                                  | Seller order masuk + detail + chat negosiasi 3 pihak                              |
| `/buyer`                                                                 | Dashboard Buyer (summary + recent orders + quick links)                           |
| `/buyer/orders`, `/buyer/orders/[id]`, `/buyer/orders/new`               | Buyer list order, detail + chat + cancel, checkout cart                           |
| `/admin/categories`                                                      | Admin kelola kategori                                                             |
| `/admin/stores`                                                          | Admin verifikasi toko                                                             |

> **Seeder demo:** `cd eksportir-api && npm run seed:sellers` → 4 seller (semua pw `seller123`: budi/wayan/made/kadek @eksportir.com) masing-masing 5 produk verified. Foto = picsum (data bohongan).

### URL penting

- API: `http://localhost:3001/api`
- **Scalar API Docs: `http://localhost:3001/docs`** (sudah aktif sejak Phase 0; endpoint baru otomatis terdokumentasi via decorator `@ApiTags`/`@ApiOperation`)
- Frontend: `http://localhost:3000`

### Arsitektur data-fetching Frontend

- **SWR + Axios** (BUKAN React Query). GET pakai `useSWR` + fetcher axios; mutasi pakai `api.post/patch/delete` lalu `mutate()`. Selalu import instance `@/lib/axios`, jangan bikin axios baru.
- Belum ada global auth state / middleware proteksi route — halaman protected mengandalkan token di `localStorage` + interceptor 401 (akan dirapikan di phase auth state).

> ⚠️ **Agent Skills:** project punya `next-tailwind-ui/.agents/skills/` (`vercel-react-best-practices`, `tailwind-design-system`). Wajib dibaca & diikuti saat menulis/refactor kode React/Next.

---

## Keputusan Upload File (penting)

Upload **dipisah dari submit data** — bukan multipart sekaligus. Pola **upload-first + staging→commit**:

```
1. FE upload file   → POST /api/uploads (multipart, 1 file)
                      → BE validasi (size 2MB, MIME, magic bytes)
                      → simpan ke R2 folder tmp/  → return { url: ".../tmp/uuid.jpg" }
2. FE isi form      → POST /api/products (JSON murni, sertakan url dari step 1)
                      → BE storage.commitFile(tmpUrl, 'products')
                        = copy tmp/uuid.jpg → products/uuid.jpg, lalu delete tmp/uuid.jpg
                      → URL FINAL (products/...) yang disimpan ke DB, bukan tmp/
3. Orphan cleanup   → cron harian 03:00 hapus tmp/ yang > 24 jam (file tak ter-commit)
```

**Alasan keputusan:**

- Upload terpisah → request data tetap JSON murni; user bisa upload sambil isi form; bisa preview sebelum submit; retry granular.
- Validasi **hanya** di Backend yang mengikat (FE bisa di-bypass via curl/Postman). Magic-bytes dicek karena MIME header bisa dipalsukan.
- File **tidak pernah dieksekusi** di BE — hanya buffer di RAM lalu stream ke R2 (object storage tidak execute apapun). Aman dari "malicious file".
- Staging `tmp/` → commit memudahkan cleanup orphan (cukup hapus semua `tmp/` lama). Di R2 tidak ada "move", jadi commit = copy + delete object.

**API StorageService:** `uploadTemp(file)`, `commitFile(tmpUrl, destFolder)`, `cleanupTemp(maxAgeHours)`, plus `uploadFile`/`deleteFile` lama.
**Penting:** entitas yang punya foto WAJIB panggil `commitFile` saat create/update; simpan URL hasil commit, jangan URL `tmp/`.

---

## Tech Stack

| Layer           | Teknologi                                              |
| --------------- | ------------------------------------------------------ |
| Frontend        | Next.js 16, Tailwind CSS v4, TypeScript                |
| Backend         | NestJS 11, TypeScript, `"type": "module"` (ESM)        |
| Database        | PostgreSQL 17 via Docker + Prisma ORM v7               |
| Auth            | JWT (Access Token) — 3 endpoint terpisah per user type |
| File Storage    | Cloudflare R2 (S3-compatible)                          |
| Validation      | class-validator + class-transformer                    |
| API Client (FE) | Axios (`lib/axios.ts`) + SWR                           |

---

## Catatan Penting Prisma 7

Prisma 7 mengubah cara koneksi database secara signifikan:

- **`url` TIDAK lagi ada di `schema.prisma`** — URL hanya di `prisma.config.ts` (untuk CLI) dan adapter (untuk runtime)
- **Wajib pakai `@prisma/adapter-pg`** untuk koneksi runtime:
  ```typescript
  import { PrismaPg } from '@prisma/adapter-pg'
  const adapter = new PrismaPg(process.env.DATABASE_URL!)
  super({ adapter })
  ```
- **`"type": "module"`** wajib ada di `package.json` karena generated client Prisma 7 menggunakan ESM (`import.meta.url`)
- Generated client ada di `generated/prisma/client.ts` — import dengan path `../../generated/prisma/client.js`
- Seed dikonfigurasi di `prisma.config.ts` bukan di `package.json`

---

## Setup Lokal

> ⚠️ **Prisma migrate wajib via Telkomsel.** `prisma migrate deploy` / `prisma db seed` ke Supabase hanya berhasil dari jaringan Telkomsel. Koneksi ISP/WiFi lain konsisten gagal (diduga routing/blokir port ke Supabase AP-Southeast). Pastikan hotspot Telkomsel aktif saat migrate.

### Menjalankan Backend

```bash
cd eksportir-api
# Tidak perlu docker compose — DB ada di Supabase
npm install
npx prisma migrate deploy   # wajib jaringan Telkomsel
npx prisma db seed          # wajib jaringan Telkomsel
npm run start:dev
```

### Database — Supabase (sejak 2026-07-12)

- `DATABASE_URL` → pooler port **6543** (`?pgbouncer=true`) — runtime app (PrismaService).
- `DIRECT_URL` → pooler port **5432** — Prisma CLI (migrate/seed/introspect).
- Docker PostgreSQL lokal sudah dihapus. `docker-compose.yml` di VPS tidak lagi punya service `postgres`.

### Akun Default (Development)

| Role       | Email                    | Password      | Login Endpoint             |
| ---------- | ------------------------ | ------------- | -------------------------- |
| SuperAdmin | superadmin@eksportir.com | superadmin123 | POST /api/auth/admin/login |

### URL Lokal

- API: `http://localhost:3001/api`
- Scalar Docs: `http://localhost:3001/docs`
- Frontend: `http://localhost:3000`

---

## Keputusan Arsitektur Database

### User — 3 Table Terpisah

Bukan 1 table `User` dengan kolom `role`. Alasan: concern keamanan dan separation of concern antara entitas yang fundamentally berbeda.

```
Buyer       → self-register, bebas daftar
Seller      → self-register, butuh verifikasi admin sebelum aktif
AdminUser   → TIDAK ada self-register, dibuat langsung oleh SuperAdmin
```

**Catatan penting:**

- 1 orang TIDAK bisa jadi Seller sekaligus Buyer dengan 1 akun — harus beda akun
- Boleh menggunakan nomor telepon yang sama di akun Buyer dan Seller
- `AdminUser` memiliki field `role` enum: `ADMIN | SUPER_ADMIN`

### Auth — 3 Endpoint Login Terpisah

```
POST /auth/buyer/login
POST /auth/seller/login
POST /auth/admin/login    ← dipakai oleh ADMIN dan SUPER_ADMIN
```

JWT payload menyimpan: `{ sub: id, userType: "buyer" | "seller" | "admin" }`
Backend menggunakan `userType` untuk menentukan table mana yang di-lookup saat validasi token.
Admin vs SuperAdmin dibedakan dari field `role` di response, bukan dari endpoint yang berbeda.

### Admin Creation Flow

- **Tidak ada invitation email, tidak ada inviteToken**
- SuperAdmin membuat akun AdminUser dan set password langsung via endpoint
- Pemberitahuan ke Admin dilakukan secara manual (di luar sistem)
- Admin bisa ganti password via `PATCH /auth/admin/change-password`

### Store

- 1 Seller = **1 Store** (untuk saat ini, tidak multi-store)
- Store butuh verifikasi Admin sebelum produk bisa tampil publik

### Category Produk

- Dikelola oleh **Admin / SuperAdmin** saja — tidak bisa dibuat Seller
- Bukan free-text, bukan enum — melainkan **master table `Category`**

### Product & Variant

Struktur:

```
Product (parent)
├── storeId
├── nama, deskripsi
├── categoryId       → relasi ke master table Category
├── isActive
├── ProductImage[]   → gallery foto produk (bisa kosong, bisa banyak)
└── ProductVariant[] → minimal 1 variant wajib ada

ProductVariant
├── productId
├── nama             → free-text, Seller isi sendiri (contoh: "25KG", "Merah", "Size L")
├── harga            → dalam IDR (input Seller)
├── stok
├── foto             → nullable String, maksimal 1 foto per variant
└── sku              → opsional
```

**Aturan variant:**

- Variant bersifat **1 dimensi / simple** — nama free-text, tidak ada kombinasi atribut (ukuran × warna)
- Minimal **1 variant wajib** ada per produk
- Foto variant nullable — kalau kosong, FE yang handle fallback
- Kebanyakan Seller UMKM Bali akan membuat **1 produk = 1 variant** ("Default") — sistem harus tetap simpel, jangan paksa UI yang ribet
- Kalau Seller hanya punya 1 jenis, cukup isi nama variant "Default" atau nama produknya langsung

**Foto produk:**

- `ProductImage` — untuk gallery di halaman produk (carousel di FE), bisa kosong
- `ProductVariant.foto` — 1 foto per variant, nullable

---

## Keputusan Mata Uang & Pembayaran

### Alur Lengkap

```
Seller input harga listing  → IDR
Negosiasi & harga final     → USD (dikunci Admin setelah deal)
Buyer bayar                 → USDT (TRC20 / ERC20)
                              *Alasan: Rusia diblokir dari SWIFT*
Platform terima             → USDT (escrow di wallet platform)
Platform konversi           → USDT → IDR (kurs saat pencairan)
Seller menerima             → IDR via transfer bank lokal
```

### Schema Transaction (penting untuk audit trail)

```
Transaction
├── amountUsdt        → USDT yang diterima dari Buyer
├── amountUsd         → equivalent USD (≈ 1:1 dengan USDT)
├── usdtToIdrRate     → kurs USDT/IDR saat konversi
├── amountIdr         → IDR yang ditransfer ke Seller
├── platformFeeUsdt   → fee platform (jika ada)
├── walletAddress     → alamat wallet platform penerima
├── txHash            → transaction hash blockchain (bukti bayar)
├── network           → "TRC20" | "ERC20"
├── status            → PENDING | CONFIRMED | RELEASED | REFUNDED
└── disbursedAt       → timestamp pencairan ke Seller
```

Verifikasi pembayaran menggunakan **txHash di blockchain explorer**, bukan screenshot transfer bank.

### SellerProfile — Info Rekening Bank

```
SellerProfile
├── bankName
├── bankAccountNumber
├── bankAccountName
└── ...
```

Dibutuhkan saat platform mentransfer IDR ke Seller setelah transaksi selesai.

---

## Alur Negosiasi (Order Flow)

Order di platform ini **tidak langsung fixed price** — ada ruang negosiasi setelah order request masuk.

```
1. Seller listing produk (harga awal = asking price dalam IDR)
2. Buyer tertarik → buat Order Request
3. Ruang chat terbuka: Buyer + Seller + Admin (3 pihak)
4. Negosiasi harga, kuantitas, syarat pengiriman
5. Admin "lock" deal → harga final dikunci dalam USD
6. Buyer transfer USDT → escrow platform (submit txHash)
7. Admin assign QC
8. QC lolos → dokumen ekspor disiapkan
9. Barang dikirim
10. Pengiriman dikonfirmasi → platform konversi USDT → IDR → transfer ke Seller
```

Perlu table `OrderMessage` untuk menyimpan chat 3 pihak per order.

---

## Keputusan Internasionalisasi (i18n)

### Library

`next-intl` — untuk Next.js App Router, locale-based routing.

### Bahasa per Area

| Area                                 | Bahasa  | Default |
| ------------------------------------ | ------- | ------- |
| Public pages (landing, produk, toko) | EN + RU | EN      |
| Buyer dashboard                      | EN + RU | EN      |
| Seller dashboard                     | EN + ID | EN      |
| Admin dashboard                      | EN + ID | EN      |

### Struktur URL

```
/en/products        ← English (public & buyer)
/ru/products        ← Russian (public & buyer)
/seller/...         ← Seller dashboard (EN/ID, tanpa locale prefix di URL)
/admin/...          ← Admin dashboard (EN/ID, tanpa locale prefix di URL)
```

### Aturan Konten

- **UI elements** (tombol, label, navbar, footer, alert, warning, form placeholder) → wajib diterjemahkan
- **Konten produk** (nama, deskripsi, nama varian) → TIDAK diterjemahkan, tetap bahasa Seller (ID/EN/Bali)
- **Konten chat/negosiasi** → TIDAK diterjemahkan, apa adanya
- Translation file ada di `next-tailwind-ui/messages/en.json` dan `ru.json` (dan `id.json` untuk seller/admin)

---

## Keputusan Currency Display

### Prinsip Utama

- **Harga listing Seller** → IDR (input Seller)
- **Harga deal dikunci** → USD (Admin lock setelah negosiasi)
- **Actual payment** → USDT (TRC20/ERC20)
- **Display ke Buyer** → USD (default), bisa switch ke RUB atau CNY (hanya tampilan, bukan pembayaran)

RUB dan CNY adalah **purely cosmetic** — untuk kenyamanan browsing Buyer dari pasar Rusia/CIS.
CNY disertakan karena Buyer Rusia kemungkinan menggunakan Yuan sebagai jalur konversi ke USDT (akibat sanksi SWIFT).
Selalu tampilkan disclaimer kecil: **"Prices in RUB/CNY are indicative. Actual payment in USDT."**

### Sumber Kurs

Kurs disimpan di DB. **SuperAdmin input manual dulu** — cron job otomatis dipikirkan belakangan.
Admin dapat **memantau** kurs terkini di dashboard.
**SuperAdmin** dapat melakukan **override manual** kurs kapan saja (otorisasi tertinggi).
Pasangan kurs yang dikelola: `USD→RUB`, `USD→CNY`, `USDT→IDR`.

> Rencana ke depan: cron job dari frankfurter.app (fiat) + CoinGecko (USDT→IDR), keduanya gratis.
> SuperAdmin tetap bisa override manual bahkan setelah cron aktif.

### Schema `ExchangeRate`

```
ExchangeRate
├── id
├── fromCurrency  → "USD" | "USDT"
├── toCurrency    → "RUB" | "CNY" | "IDR"
├── rate          → Decimal (presisi tinggi)
├── source        → "AUTO" | "MANUAL" (siapa yang set)
├── updatedBy     → nullable, AdminUser.id (kalau manual)
└── updatedAt     → timestamp
```

---

## Hal-hal yang Belum Diputuskan

- [ ] Fee platform — berapa persen, siapa yang menanggung?
- [ ] Apakah ada fitur rating/review Seller oleh Buyer?
- [ ] Refund flow — kalau QC gagal, USDT dikembalikan ke Buyer?
- [ ] Notifikasi — realtime (WebSocket) atau polling?
- [ ] Sumber API kurs otomatis (Fixer.io, ExchangeRate-API, atau lainnya)?
- [ ] Interval cron job update kurs (setiap jam, setiap 6 jam?)?

---

## Konvensi Kode

### Backend (NestJS)

- Semua import file lokal wajib pakai ekstensi `.js` (karena ESM + `nodenext` module resolution)
  ```typescript
  import { AuthService } from './auth.service.js'  // ✅
  import { AuthService } from './auth.service'      // ❌
  ```
- Gunakan `ConfigService` untuk semua env var, jangan akses `process.env` langsung di service/controller
- Response error dari `ValidationPipe` adalah array — handle di FE dengan `Array.isArray(msg)`
- CORS dikonfigurasi di `main.ts` — tambahkan origin baru di sini saat deploy ke staging/production

### Frontend (Next.js)

- Gunakan `api` instance dari `@/lib/axios` — **jangan buat axios instance baru**
- Token disimpan di `localStorage` key `accessToken` dan `user`
- Response interceptor di `lib/axios.ts` otomatis redirect ke `/admin/login` kalau 401
- Gunakan SWR dengan axios fetcher untuk semua GET request:
  ```typescript
  const fetcher = (url: string) => api.get(url).then(r => r.data)
  const { data } = useSWR('/api/users/me', fetcher)
  ```
- Selalu pakai design token dari `globals.css` — jangan hardcode warna hex
- Komponen ada di `stories/` — import langsung, jangan buat komponen baru kalau sudah ada
