
docker compose build --no-cache && docker compose up -d

docker compose build --pull --no-cache && docker compose up -d

docker-compose up -d        → PostgreSQL jalan, DB "eksportir" dibuat kosong
npx prisma migrate deploy   → Prisma masuk ke DB, buat semua tabel
npx prisma db seed          → isi data awal (SuperAdmin, dll)