# Allo Engineering - Inventory Reservation System

A production-ready inventory reservation system for multi-warehouse retail that prevents overselling during payment processing (3DS, UPI, wallet redirects) by implementing time-based stock reservations with concurrency guarantees.

## Live Demo

[Insert your Vercel deployment URL here]

## Features

- ✅ **Race-condition-free reservations** - Distributed locking ensures exactly one customer can reserve the last unit
- ✅ **Real-time stock tracking** - Available stock = total units - reserved units
- ✅ **Automatic expiry** - Unconfirmed reservations release automatically after 10 minutes
- ✅ **Idempotent API** (Bonus) - Safe retries with Idempotency-Key headers
- ✅ **Live countdown timer** - Frontend shows remaining time before reservation expires
- ✅ **Optimistic UI updates** - No full page refreshes after confirm/cancel actions

## Tech Stack

- **Framework**: Next.js 14 (App Router)
- **Language**: TypeScript
- **Database**: PostgreSQL (Supabase)
- **ORM**: Prisma
- **Caching & Locking**: Redis (Upstash)
- **Validation**: Zod
- **Styling**: Tailwind CSS + shadcn/ui
- **Deployment**: Vercel (app) + Supabase (DB) + Upstash (Redis)

## Architecture Overview

### Core Problem
When payment takes minutes (3DS flows, UPI confirmations), two customers can pay for the same physical unit if we decrement stock only at payment time. If we decrement at cart, 80% abandonment kills conversion.

### Solution: Time-based Reservations
1. Customer proceeds to checkout → Reserve units for 10 minutes
2. Available stock = total - reserved (atomic operation)
3. Payment succeeds → Confirm reservation (permanently decrement)
4. Payment fails/times out → Release reservation (return to available)

### Concurrency Guarantee
The critical section is checking stock availability and creating a reservation atomically. We solve this using:

**Redis Distributed Lock** (Redlock algorithm):

Key: lock:product:{productId}:warehouse:{warehouseId}
TTL: 5 seconds (prevents deadlocks)

This ensures only one request can modify a specific product-warehouse stock at a time.

## API Endpoints

| Method | Path | Behavior |
|--------|------|----------|
| GET | `/api/products` | List products with available stock per warehouse |
| GET | `/api/warehouses` | List all warehouses |
| POST | `/api/reservations` | Reserve units (returns 409 if insufficient stock) |
| POST | `/api/reservations/:id/confirm` | Confirm reservation (returns 410 if expired) |
| POST | `/api/reservations/:id/release` | Manually release reservation |

### Idempotency (Bonus)
All write endpoints support `Idempotency-Key` header:
- Store successful responses in Redis for 24 hours
- Retried requests return original response without re-executing
- Prevents duplicate reservations/payments

## Database Schema

```prisma
model Product {
  id        String    @id @default(cuid())
  sku       String    @unique
  name      String
  price     Float
  stocks    Stock[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Warehouse {
  id        String    @id @default(cuid())
  name      String
  location  String
  stocks    Stock[]
  createdAt DateTime  @default(now())
  updatedAt DateTime  @updatedAt
}

model Stock {
  id         String    @id @default(cuid())
  productId  String
  warehouseId String
  totalUnits Int       @default(0)
  reservedUnits Int    @default(0)
  product    Product   @relation(fields: [productId], references: [id])
  warehouse  Warehouse @relation(fields: [warehouseId], references: [id])
  
  @@unique([productId, warehouseId])
}

model Reservation {
  id          String    @id @default(cuid())
  productId   String
  warehouseId String
  quantity    Int
  status      ReservationStatus @default(PENDING)
  expiresAt   DateTime
  confirmedAt DateTime?
  releasedAt  DateTime?
  product     Product   @relation(fields: [productId], references: [id])
  warehouse   Warehouse @relation(fields: [warehouseId], references: [id])
  createdAt   DateTime  @default(now())
  updatedAt   DateTime  @updatedAt
  
  @@index([status, expiresAt])
}

enum ReservationStatus {
  PENDING
  CONFIRMED
  RELEASED
}

model IdempotencyKey {
  id         String   @id @default(cuid())
  key        String   @unique
  response   Json
  createdAt  DateTime @default(now())
  expiresAt  DateTime
  
  @@index([expiresAt])
}
