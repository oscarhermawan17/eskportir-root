# Eksportir — Platform Perdagangan Ekspor B2B

> Menghubungkan penjual dari Indonesia (fokus Bali) dengan pembeli internasional, khususnya di Rusia dan negara-negara CIS.

---

## Gambaran Umum

**Eksportir** adalah platform commerce B2B yang dirancang untuk memfasilitasi proses ekspor barang dari Indonesia ke pasar internasional. Berbeda dengan platform e-commerce biasa, Eksportir beroperasi sebagai **marketplace terkelola** di mana platform berperan sebagai perantara terpercaya — menangani negosiasi, pengecekan kualitas, dokumentasi, penahanan dana (escrow), dan koordinasi logistik.

Platform ini dibangun untuk **transaksi bernilai tinggi dan volume besar** di mana kepercayaan, dokumentasi, dan kepatuhan regulasi adalah hal yang kritis.

---

## Fitur Utama

- 🏪 **Dashboard Seller** — Seller dapat mendaftar, mengelola toko, dan mendaftarkan produk
- 🌍 **Portal Buyer** — Buyer internasional dapat menemukan dan mengajukan permintaan produk
- 🤝 **Alur Transaksi Terkelola** — Setiap transaksi diawasi oleh admin platform
- ✅ **Proses Quality Check (QC)** — Inspeksi fisik sebelum konfirmasi pengiriman
- 📄 **Dokumentasi Ekspor** — Pengelolaan digital untuk bea cukai, invoice, dan dokumen pengiriman
- 💰 **Escrow Dana** — Dana buyer ditahan dengan aman hingga pengiriman dikonfirmasi
- 📦 **Koordinasi Logistik** — Platform bertindak sebagai penghubung kurir ke bandara/pelabuhan
- 👤 **Sistem Multi-Role** — Seller, Buyer, Admin, Super Admin
- 🔔 **Notifikasi & Update Status** — Status transaksi real-time untuk semua pihak

---

## Tech Stack

### Frontend
| Teknologi | Kegunaan |
|-----------|---------|
| [Next.js](https://nextjs.org/) | Framework React dengan dukungan SSR/SSG |
| Tailwind CSS | Styling berbasis utility class |
| TypeScript | Type safety di seluruh codebase |

### Backend
| Teknologi | Kegunaan |
|-----------|---------|
| [NestJS](https://nestjs.com/) | Framework Node.js yang terstruktur dan scalable |
| TypeScript | Type safety dan kemudahan maintenance |
| REST API | Komunikasi antara FE dan BE |
| JWT (Access + Refresh Token) | Autentikasi & otorisasi |

### Database & Storage
| Teknologi | Kegunaan |
|-----------|---------|
| [PostgreSQL](https://www.postgresql.org/) | Database relasional utama |
| [Prisma ORM](https://www.prisma.io/) | Akses database type-safe & manajemen migrasi |
| [Cloudflare R2 (kompatibel S3)](https://developers.cloudflare.com/r2/) | Penyimpanan file & dokumen (dokumen ekspor, foto produk) |

---

## Arsitektur Sistem

```
┌─────────────────────┐        ┌─────────────────────┐
│  Frontend Next.js   │ ──────▶│   Backend NestJS     │
│  (Portal Seller/    │  REST  │   (API + Logika      │
│   Buyer / Admin)    │◀────── │    Bisnis + Auth)    │
└─────────────────────┘        └──────────┬──────────┘
                                          │
                    ┌─────────────────────┼──────────────────┐
                    │                     │                  │
           ┌────────▼──────┐   ┌──────────▼──────┐  ┌───────▼──────┐
           │  PostgreSQL   │   │  Cloudflare R2  │  │  Cron Jobs   │
           │  (Prisma ORM) │   │  (Dokumen &     │  │  (Tugas      │
           │               │   │   Foto Produk)  │  │  Terjadwal)  │
           └───────────────┘   └─────────────────┘  └──────────────┘
```

---

## Alur Transaksi

```
1. Seller mendaftarkan produk
        │
2. Buyer mengajukan permintaan pembelian
        │
3. Admin meninjau & memfasilitasi negosiasi
        │
4. Buyer menyetor dana (escrow)
        │
5. Quality Check (QC) oleh tim platform
        │
6. Dokumen ekspor disiapkan
        │
7. Barang diserahkan ke logistik / bandara
        │
8. Pengiriman dikonfirmasi → dana dicairkan ke Seller
```

---

## Sistem Role

| Role | Kemampuan |
|------|----------|
| **Seller** | Kelola toko, daftarkan produk, pantau pesanan |
| **Buyer** | Telusuri produk, ajukan pesanan, pantau pengiriman |
| **Admin** | Kelola transaksi, proses QC, dokumentasi |
| **Super Admin** | Kontrol penuh platform, manajemen pengguna, pelaporan |

---

## Struktur Project

```
eksportir/
├── next-tailwind-ui/        # Frontend Next.js
│   ├── app/                 # Halaman dengan App Router
│   ├── stories/             # Komponen UI
│   └── public/
│
└── eksportir-api/           # Backend NestJS
    ├── src/
    │   ├── auth/            # Autentikasi JWT
    │   ├── users/           # Manajemen pengguna
    │   ├── products/        # Listing produk
    │   ├── orders/          # Alur pesanan & transaksi
    │   ├── documents/       # Dokumentasi ekspor
    │   ├── storage/         # Integrasi Cloudflare R2
    │   └── admin/           # Modul Admin & Super Admin
    ├── prisma/
    │   └── schema.prisma    # Skema database
    └── .env
```

---

## Cara Menjalankan

### Prasyarat
- Node.js >= 20
- PostgreSQL >= 15
- Bucket Cloudflare R2 (atau storage kompatibel S3 lainnya)

### Setup Backend

```bash
cd eksportir-api
npm install
cp .env.example .env
# Isi variabel environment yang dibutuhkan

npx prisma migrate dev
npm run start:dev
```

### Setup Frontend

```bash
cd next-tailwind-ui
npm install
cp .env.local.example .env.local
# Isi variabel environment yang dibutuhkan

npm run dev
```

---

## Variabel Environment

### Backend (`.env`)
```env
DATABASE_URL=postgresql://user:password@localhost:5432/eksportir
JWT_SECRET=your_jwt_secret
JWT_REFRESH_SECRET=your_refresh_secret
JWT_EXPIRES_IN=15m
JWT_REFRESH_EXPIRES_IN=7d

R2_ACCOUNT_ID=your_cloudflare_account_id
R2_ACCESS_KEY_ID=your_r2_access_key
R2_SECRET_ACCESS_KEY=your_r2_secret_key
R2_BUCKET_NAME=eksportir-docs
R2_PUBLIC_URL=https://your-r2-public-url
```

### Frontend (`.env.local`)
```env
NEXT_PUBLIC_API_URL=http://localhost:3001
```

---

## Lisensi

Privat & Proprietary — Seluruh hak cipta dilindungi.

---

> Dibangun dengan ❤️ dari Bali, Indonesia 🌴
