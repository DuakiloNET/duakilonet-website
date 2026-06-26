---
title: "Pola PostgreSQL + Python: Indexing, Connection Pooling, JSONB, dan Async Query"
date: 2026-06-26
description: "Empat pola PostgreSQL yang paling sering bikin aplikasi Python lambat kalau diabaikan: strategi index yang tepat, connection pooling yang benar, JSONB sebagai pengganti EAV, dan async query dengan asyncpg/SQLAlchemy."
tags: ["postgresql", "python", "asyncpg", "sqlalchemy", "database", "performance"]
categories: ["python"]
draft: false
showHero: false
showTableOfContents: true
showReadingTime: true
---

Hampir semua bug performa di aplikasi Python yang pakai PostgreSQL berakar dari empat hal: index yang salah (atau tidak ada), connection yang dibuka-tutup tanpa pooling, struktur data semi-relasional yang dipaksa jadi tabel kaku, dan query sync yang menyumbat event loop async. Tulisan ini membahas keempatnya dengan contoh kode konkret — bukan teori abstrak soal database tuning.

## 1. Indexing: Lebih dari Sekadar `CREATE INDEX`

Index yang salah sama buruknya dengan tidak ada index — query planner tetap full scan, tapi sekarang ada overhead tambahan setiap kali kamu insert/update. Tiga pola yang paling sering kepakai di aplikasi nyata:

### Index Parsial untuk Query yang Selektif

Kalau 90% query cuma butuh baris dengan `status = 'active'`, index full di kolom `status` boros — kebanyakan entri index tidak pernah dipakai. Index parsial menyimpan hanya baris yang relevan:

```sql
CREATE INDEX idx_orders_active
ON orders (created_at)
WHERE status = 'active';
```

Index ini lebih kecil, lebih cepat di-scan, dan tetap valid dipakai planner asal query-nya juga punya `WHERE status = 'active'`.

### Composite Index — Urutan Kolom Penting

```sql
CREATE INDEX idx_orders_user_created
ON orders (user_id, created_at DESC);
```

Urutan kolom menentukan index ini bisa dipakai untuk query apa. Index `(user_id, created_at)` efektif untuk:

```sql
SELECT * FROM orders WHERE user_id = 42 ORDER BY created_at DESC;
```

tapi tidak membantu sama sekali untuk query yang filter `created_at` saja tanpa `user_id` — PostgreSQL tidak bisa "skip" kolom pertama composite index. Aturan praktis: kolom dengan equality filter (`=`) di depan, kolom untuk sorting/range di belakang.

### Cek Index Beneran Dipakai

Jangan asumsi — verifikasi dengan `EXPLAIN ANALYZE`:

```python
import psycopg

async def check_query_plan(conn: psycopg.AsyncConnection, query: str):
    async with conn.cursor() as cur:
        await cur.execute(f"EXPLAIN ANALYZE {query}")
        rows = await cur.fetchall()
        for row in rows:
            print(row[0])
```

Kalau muncul `Seq Scan` di tabel besar padahal kamu sudah bikin index, biasanya karena: planner mengira full scan lebih murah (statistik kolom stale — jalankan `ANALYZE orders`), atau index-nya tidak match kondisi `WHERE` (misal pakai fungsi di kolom tanpa expression index).

### Index untuk Fungsi/Expression

```sql
-- query ini tidak akan pakai index biasa di kolom email
SELECT * FROM users WHERE lower(email) = 'budi@example.com';

-- bikin expression index yang match
CREATE INDEX idx_users_email_lower ON users (lower(email));
```

## 2. Connection Pooling: Jangan Buka Koneksi per Request

Membuka koneksi PostgreSQL baru itu mahal — ada handshake TCP, autentikasi, alokasi memori di sisi server. Kalau aplikasi web buka-tutup koneksi setiap request, latency naik dan server Postgres cepat kehabisan `max_connections`.

### Pooling di Level Aplikasi (asyncpg)

```python
import asyncpg

pool: asyncpg.Pool | None = None

async def init_pool():
    global pool
    pool = await asyncpg.create_pool(
        dsn="postgresql://user:pass@localhost/mydb",
        min_size=5,
        max_size=20,
        max_inactive_connection_lifetime=300,
    )

async def get_user(user_id: int) -> dict | None:
    async with pool.acquire() as conn:
        row = await conn.fetchrow(
            "SELECT id, email, name FROM users WHERE id = $1", user_id
        )
        return dict(row) if row else None
```

`min_size`/`max_size` menentukan rentang koneksi yang dijaga pool. Set `max_size` terlalu besar justru kontraproduktif — tiap koneksi Postgres makan memori server (sekitar 5-10MB per koneksi tergantung `work_mem`), jadi 20 connection per worker process kali 4 worker bisa cepat mendekati limit `max_connections` default (100).

### Pooling di Level Infrastruktur (PgBouncer)

Kalau aplikasi punya banyak worker process (Gunicorn dengan beberapa worker, masing-masing dengan pool sendiri), total koneksi ke Postgres bisa meledak. PgBouncer jadi proxy yang multipleks banyak koneksi aplikasi ke jumlah koneksi backend yang lebih kecil:

```ini
[databases]
mydb = host=localhost port=5432 dbname=mydb

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000
default_pool_size = 25
```

`pool_mode = transaction` paling umum untuk web app — koneksi backend dilepas balik ke pool begitu transaksi selesai, bukan ditahan sepanjang session client. Trade-off: fitur session-level seperti `SET` per-session atau prepared statement lintas-transaksi tidak reliable di mode ini, jadi cek dulu driver/ORM kamu kompatibel.

### Anti-Pattern yang Sering Kejadian

```python
# JANGAN — bikin koneksi baru tiap function call
async def get_user_bad(user_id: int):
    conn = await asyncpg.connect(dsn="...")
    row = await conn.fetchrow("SELECT * FROM users WHERE id = $1", user_id)
    await conn.close()
    return row
```

Kalau endpoint ini dipanggil 100 request/detik, itu 100 koneksi baru per detik — overhead handshake saja sudah jadi bottleneck sebelum query-nya dieksekusi.

## 3. JSONB: Fleksibel, Tapi Tetap Bisa Diindex

JSONB sering dipakai untuk data yang strukturnya bervariasi — metadata produk, custom field, event payload. Kelebihannya dibanding EAV (entity-attribute-value) klasik: tidak perlu join berlapis, dan tetap bisa diquery + diindex secara native.

### Simpan dan Query JSONB

```python
import json

async def save_event(pool: asyncpg.Pool, event_type: str, payload: dict):
    await pool.execute(
        "INSERT INTO events (event_type, payload) VALUES ($1, $2)",
        event_type,
        json.dumps(payload),
    )

async def find_events_by_field(pool: asyncpg.Pool, key: str, value: str):
    return await pool.fetch(
        "SELECT * FROM events WHERE payload->>$1 = $2",
        key, value,
    )
```

`->>` mengekstrak nilai sebagai text, `->` mengekstrak sebagai JSON (berguna kalau nested). Untuk filter yang lebih kompleks, operator `@>` (containment) sering lebih efisien:

```sql
SELECT * FROM events WHERE payload @> '{"status": "failed", "retried": true}';
```

### Index GIN untuk JSONB

Tanpa index, query JSONB di tabel besar full scan dan parse JSON per baris — mahal. Index GIN menyelesaikan ini:

```sql
CREATE INDEX idx_events_payload ON events USING GIN (payload);
```

Index ini mengakselerasi operator containment (`@>`, `?`, `?&`, `?|`). Kalau query-mu konsisten cuma akses satu key tertentu, expression index lebih hemat space:

```sql
CREATE INDEX idx_events_status ON events ((payload->>'status'));
```

### Kapan JSONB, Kapan Kolom Biasa?

Aturan praktis: kalau sebuah field dipakai untuk filter/join/sort di hampir semua query, promosikan jadi kolom biasa dengan tipe dan index yang sesuai — JSONB kalah cepat dibanding kolom native untuk kasus ini. JSONB paling masuk akal untuk data yang strukturnya genuinely variabel antar baris (custom attribute per tenant, payload webhook dari pihak ketiga) atau data yang jarang diquery langsung, cuma disimpan untuk audit/debug.

## 4. Async Query: SQLAlchemy 2.0 + asyncpg

Kalau aplikasi pakai FastAPI atau framework async lain, query sync (psycopg2 biasa) memblokir event loop selama I/O database berlangsung — request lain di worker yang sama ikut tertahan. SQLAlchemy 2.0 dengan driver `asyncpg` menyelesaikan ini.

### Setup Engine Async

```python
from sqlalchemy.ext.asyncio import create_async_engine, AsyncSession
from sqlalchemy.orm import sessionmaker

engine = create_async_engine(
    "postgresql+asyncpg://user:pass@localhost/mydb",
    pool_size=10,
    max_overflow=5,
    pool_pre_ping=True,
)

AsyncSessionLocal = sessionmaker(
    engine, class_=AsyncSession, expire_on_commit=False
)
```

`pool_pre_ping=True` penting di production — mengecek koneksi masih hidup sebelum dipakai, supaya koneksi yang putus karena idle timeout (PgBouncer atau firewall) tidak bikin query gagal dengan error yang membingungkan.

### Query dengan AsyncSession

```python
from sqlalchemy import select
from myapp.models import User

async def get_active_users(session: AsyncSession, limit: int = 50) -> list[User]:
    result = await session.execute(
        select(User).where(User.status == "active").limit(limit)
    )
    return list(result.scalars().all())
```

### Pattern Dependency Injection di FastAPI

```python
from fastapi import Depends

async def get_session() -> AsyncSession:
    async with AsyncSessionLocal() as session:
        yield session

@app.get("/users/active")
async def list_active_users(session: AsyncSession = Depends(get_session)):
    users = await get_active_users(session)
    return [{"id": u.id, "email": u.email} for u in users]
```

### Jangan Campur Sync dan Async Driver

Kesalahan umum: pakai `psycopg2` (sync) di dalam endpoint async tanpa `run_in_executor`. Ini terlihat jalan saat testing dengan load rendah, tapi di production dengan concurrent request tinggi, setiap query sync memblokir seluruh event loop — efeknya mirip single-threaded server yang antri satu-satu walau endpoint-nya ditulis `async def`.

```python
# JANGAN — psycopg2 sync di endpoint async, memblokir event loop
@app.get("/bad-example")
async def bad_example():
    conn = psycopg2.connect(...)  # blocking call di async context
    cur = conn.cursor()
    cur.execute("SELECT * FROM users")
    return cur.fetchall()
```

Kalau terpaksa harus pakai library sync (driver lama, atau ORM yang belum support async), bungkus lewat thread pool:

```python
import asyncio
from functools import partial

async def query_with_sync_driver(query: str):
    loop = asyncio.get_event_loop()
    return await loop.run_in_executor(None, partial(run_sync_query, query))
```

Tapi ini solusi sementara — kalau project baru dan punya pilihan, asyncpg/psycopg3 async langsung lebih bersih daripada menambah layer thread pool.

## Checklist Singkat

- **Index**: cek `EXPLAIN ANALYZE` sebelum percaya index dipakai. Composite index — equality dulu, range/sort belakangan. Index parsial untuk subset data yang sering diquery.
- **Pooling**: jangan buka koneksi per request. Pakai pool di level aplikasi (asyncpg, SQLAlchemy) plus PgBouncer kalau worker process-nya banyak.
- **JSONB**: index GIN untuk containment query, expression index untuk akses key tunggal yang konsisten. Promosikan ke kolom biasa kalau field-nya jadi krusial untuk filter/join.
- **Async**: pakai driver async (asyncpg/psycopg3) konsisten dari engine sampai query. Sync driver di endpoint async adalah cara paling gampang membuat aplikasi "async" yang sebenarnya tidak concurrent sama sekali.

## Kesimpulan

Keempat pola ini saling berkaitan — connection pool yang sehat tidak banyak membantu kalau query-nya masih full scan tanpa index, dan async query secepat apapun tetap menunggu kalau pool kehabisan koneksi. Yang paling penting: ukur dulu sebelum optimasi. `EXPLAIN ANALYZE` untuk query, `pg_stat_activity` untuk koneksi aktif, dan log slow query (`log_min_duration_statement`) untuk tahu pattern mana yang sebenarnya jadi bottleneck di traffic nyata — bukan asumsi.

---

*Ada pola PostgreSQL + Python lain yang sering dipakai di project kamu? Share di komentar.*
