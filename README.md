# Eksportir — B2B Export Trading Platform

> Connecting Indonesian sellers (Bali-focused) with international buyers, particularly in Russia and CIS countries.

---

## Overview

**Eksportir** is a B2B commerce platform designed to facilitate the export of goods from Indonesia to international markets. Unlike a typical e-commerce platform, Eksportir operates as a **managed marketplace** where the platform acts as a trusted middleman — handling negotiations, quality checks, documentation, fund escrow, and logistics coordination.

This platform is built for **high-value, high-volume transactions** where trust, documentation, and compliance are critical.

---

## Key Features

- 🏪 **Seller Dashboard** — Sellers can register, manage their store, and list products
- 🌍 **Buyer Portal** — International buyers can discover and request products
- 🤝 **Managed Transaction Flow** — Every transaction is supervised by platform admins
- ✅ **Quality Check (QC) Process** — Physical inspection before shipment confirmation
- 📄 **Export Documentation** — Digital management of customs, invoices, and shipping docs
- 💰 **Fund Escrow** — Buyer funds are held securely until delivery is confirmed
- 📦 **Logistics Coordination** — Platform acts as courier liaison for airport/port handoff
- 👤 **Multi-role System** — Seller, Buyer, Admin, Super Admin
- 🔔 **Notifications & Status Updates** — Real-time transaction status for all parties

---

## Tech Stack

### Frontend
| Technology | Purpose |
|------------|---------|
| [Next.js](https://nextjs.org/) | React framework with SSR/SSG support |
| Tailwind CSS | Utility-first styling |
| TypeScript | Type safety across the codebase |

### Backend
| Technology | Purpose |
|------------|---------|
| [NestJS](https://nestjs.com/) | Structured, scalable Node.js framework |
| TypeScript | Type safety and maintainability |
| REST API | Communication between FE and BE |
| JWT (Access + Refresh Token) | Authentication & authorization |

### Database & Storage
| Technology | Purpose |
|------------|---------|
| [PostgreSQL](https://www.postgresql.org/) | Primary relational database |
| [Prisma ORM](https://www.prisma.io/) | Type-safe database access & migrations |
| [Cloudflare R2 (S3-compatible)](https://developers.cloudflare.com/r2/) | File & document storage (export docs, product images) |

---

## Architecture Overview

```
┌─────────────────────┐        ┌─────────────────────┐
│   Next.js Frontend  │ ──────▶│   NestJS Backend     │
│   (Seller/Buyer/    │  REST  │   (API + Business    │
│    Admin Portal)    │◀────── │    Logic + Auth)     │
└─────────────────────┘        └──────────┬──────────┘
                                          │
                    ┌─────────────────────┼──────────────────┐
                    │                     │                  │
           ┌────────▼──────┐   ┌──────────▼──────┐  ┌───────▼──────┐
           │  PostgreSQL   │   │  Cloudflare R2  │  │  Cron Jobs   │
           │  (Prisma ORM) │   │  (Documents &   │  │  (Scheduled  │
           │               │   │   Images)       │  │   Tasks)     │
           └───────────────┘   └─────────────────┘  └──────────────┘
```

---

## Transaction Workflow

```
1. Seller lists product
        │
2. Buyer submits purchase request
        │
3. Admin reviews & facilitates negotiation
        │
4. Buyer deposits funds (escrow)
        │
5. Quality Check (QC) by platform team
        │
6. Export documentation prepared
        │
7. Goods handed to logistics / airport
        │
8. Delivery confirmed → funds released to Seller
```

---

## Role System

| Role | Capabilities |
|------|-------------|
| **Seller** | Manage store, list products, track orders |
| **Buyer** | Browse products, submit orders, track shipment |
| **Admin** | Manage transactions, QC process, documentation |
| **Super Admin** | Full platform control, user management, reporting |

---

## Project Structure

```
eksportir/
├── next-tailwind-ui/        # Next.js Frontend
│   ├── app/                 # App Router pages
│   ├── stories/             # UI Components
│   └── public/
│
└── eksportir-api/           # NestJS Backend
    ├── src/
    │   ├── auth/            # JWT Authentication
    │   ├── users/           # User management
    │   ├── products/        # Product listings
    │   ├── orders/          # Order & transaction flow
    │   ├── documents/       # Export documentation
    │   ├── storage/         # Cloudflare R2 integration
    │   └── admin/           # Admin & Super Admin modules
    ├── prisma/
    │   └── schema.prisma    # Database schema
    └── .env
```

---

## Getting Started

### Prerequisites
- Node.js >= 20
- PostgreSQL >= 15
- Cloudflare R2 bucket (or any S3-compatible storage)

### Backend Setup

```bash
cd eksportir-api
npm install
cp .env.example .env
# Fill in your environment variables

npx prisma migrate dev
npm run start:dev
```

### Frontend Setup

```bash
cd next-tailwind-ui
npm install
cp .env.local.example .env.local
# Fill in your environment variables

npm run dev
```

---

## Environment Variables

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

## License

Private & Proprietary — All rights reserved.

---

> Built with ❤️ from Bali, Indonesia 🌴
