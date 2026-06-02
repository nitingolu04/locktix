# LockTix — Smart Ticket Booking System (STBS)

A full-stack event ticket booking app with **real-time seat maps**, **distributed seat locking**, and **checkout sessions**. Built with Next.js, Firebase Auth, Supabase (PostgreSQL + Realtime), and Redis for concurrency-safe reservations.

![Next.js](https://img.shields.io/badge/Next.js-16-black)
![React](https://img.shields.io/badge/React-19-61dafb)
![TypeScript](https://img.shields.io/badge/TypeScript-5-3178c6)
![License](https://img.shields.io/badge/License-MIT-green)

---

## Features

| Area | Details |
|------|---------|
| **Events** | Browse events on the home page and open seat selection per event |
| **Seat map** | Interactive grid (rows A–B, 20 seats each) with live status updates |
| **Seat locking** | Redis `SET NX PX` locks prevent double-booking; 60s checkout timer |
| **Realtime UI** | Supabase Realtime pushes seat updates to all connected clients |
| **Auth** | Email/password sign-up and login via Firebase Authentication |
| **Checkout** | Countdown timer, multi-seat checkout, simulated payment flow |
| **Tickets** | View booked tickets, QR codes, and PDF download |
| **Idempotency** | Duplicate payment requests are deduplicated via `bookings` table |

---

## Tech stack

| Layer | Technology |
|-------|------------|
| Framework | [Next.js 16](https://nextjs.org/) (App Router) |
| UI | React 19, Tailwind CSS 4, Lucide icons |
| Auth | [Firebase Auth](https://firebase.google.com/docs/auth) |
| Database | [Supabase](https://supabase.com/) (PostgreSQL) |
| Realtime | Supabase Realtime (`postgres_changes` on `seats`) |
| Locks / TTL | [Redis](https://redis.io/) via `ioredis` (in-memory mock if `REDIS_URL` is unset) |
| Tickets | `jspdf`, `qrcode.react`, `html-to-image` |

---

## Architecture

```mermaid
flowchart LR
    subgraph Client
        Browser[Next.js pages]
    end

    subgraph Auth
        Firebase[Firebase Auth]
    end

    subgraph API["Next.js API routes"]
        Lock[lock-seat]
        Pay[pay]
        Seats[seats]
    end

    subgraph Data
        Redis[(Redis locks)]
        Supabase[(Supabase DB)]
        Realtime[Supabase Realtime]
    end

    Browser --> Firebase
    Browser --> API
    Browser --> Realtime
    Lock --> Redis
    Lock --> Supabase
    Pay --> Redis
    Pay --> Supabase
    Seats --> Redis
    Seats --> Supabase
    Realtime --> Browser
```

**Booking flow**

1. User signs in (Firebase) and selects seats on `/seats`.
2. `POST /api/lock-seat` acquires a Redis lock (`lock:seat:{id}`) and marks the seat `locked` in Supabase.
3. User proceeds to `/checkout` — a timer shows remaining lock TTL from Redis.
4. `POST /api/pay` verifies locks, simulates payment, atomically sets seats to `booked`, and clears Redis keys.
5. Success page and **My Tickets** show confirmation; PDF/QR available for download.

---

## Prerequisites

- **Node.js** 20+
- **npm** (or pnpm / yarn)
- A [Firebase](https://console.firebase.google.com/) project with **Email/Password** auth enabled
- A [Supabase](https://supabase.com/) project
- **(Optional)** Redis — [Upstash](https://upstash.com/), local Redis, or Docker. Without `REDIS_URL`, the app uses an in-memory mock (fine for solo dev; not for multi-instance production).

---

## Quick start

### 1. Clone and install

```bash
git clone <your-repo-url>
cd STBS
npm install
```

### 2. Environment variables

Create `.env.local` in the project root:

```env
# Supabase (Project Settings → API)
NEXT_PUBLIC_SUPABASE_URL=https://your-project.supabase.co
NEXT_PUBLIC_SUPABASE_ANON_KEY=your-anon-key

# Firebase (Project Settings → General → Your apps → Web app config)
NEXT_PUBLIC_FIREBASE_API_KEY=
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN=
NEXT_PUBLIC_FIREBASE_PROJECT_ID=
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET=
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID=
NEXT_PUBLIC_FIREBASE_APP_ID=

# Redis (optional — omit to use in-memory mock)
REDIS_URL=redis://default:password@host:6379
```

| Variable | Where to find it |
|----------|------------------|
| Supabase URL & anon key | Supabase → **Project Settings** → **API** |
| Firebase config | Firebase Console → **Project settings** → **Your apps** → Web app → **SDK setup** → Config |
| `REDIS_URL` | Your Redis provider connection string |

### 3. Database setup

1. Open the Supabase **SQL Editor**.
2. Paste and run the full script in [`supabase_schema.sql`](./supabase_schema.sql).

This creates `seats` and `bookings` tables, RLS policies, seeds **40 seats** (rows A and B, 1–20), and enables Realtime on `seats`.

### 4. Firebase Authentication

1. Firebase Console → **Build** → **Authentication** → **Get started**.
2. **Sign-in method** → enable **Email/Password**.

### 5. Run the app

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000).

| Script | Command |
|--------|---------|
| Development | `npm run dev` |
| Production build | `npm run build` |
| Start production | `npm start` |
| Lint | `npm run lint` |

---

## Configuration

### Session / lock timer (default: 60 seconds)

Seat reservations expire when the Redis lock TTL runs out. Change the duration in:

**`src/app/api/lock-seat/route.ts`**

```ts
await redis.set(lockKey, userId, 'PX', 60000, 'NX'); // milliseconds
```

Examples: `120000` (2 min), `300000` (5 min), `600000` (10 min).

If you change this value, update the progress bar reference in **`src/components/Timer.tsx`** (`maxTime`) so the UI stays in sync.

### Seat price

Checkout uses **₹150 per seat** (hardcoded in `src/app/checkout/page.tsx`). Adjust `pricePerSeat` there.

---

## API routes

| Method | Route | Description |
|--------|-------|-------------|
| `GET` | `/api/seats` | List all seats; merges Redis lock state with DB |
| `GET` | `/api/seat-details?seatId=` or `?seatIds=a,b` | Lock owner and TTL per seat |
| `POST` | `/api/lock-seat` | Lock a seat for the current user (`seatId`, `userId`) |
| `POST` | `/api/unlock-seat` | Release a user-owned lock |
| `POST` | `/api/release-seat` | Release lock (alternate path) |
| `POST` | `/api/pay` | Complete booking (`seatIds`, `userId`, optional `bookingId`) |
| `GET` | `/api/my-tickets?userId=` | Booked seats for a user |

---

## Project structure

```
STBS/
├── public/                 # Event images and static assets
├── src/
│   ├── app/
│   │   ├── api/            # Server routes (seats, lock, pay, …)
│   │   ├── checkout/       # Payment + session timer
│   │   ├── login|signup/   # Firebase auth pages
│   │   ├── seats/          # Seat selection + Realtime
│   │   ├── my-tickets/     # Booking history
│   │   └── …
│   ├── components/         # SeatGrid, Timer, Navbar, QRCodeModal
│   ├── contexts/           # AuthContext
│   ├── hooks/              # useAuth, useUser
│   └── lib/
│       ├── db.ts           # Supabase seat/booking operations
│       ├── redis.ts        # Redis client or in-memory mock
│       ├── firebase.ts     # Firebase init
│       ├── supabase.ts     # Supabase client
│       └── ticketDownload.ts
├── supabase_schema.sql     # Database + seed + Realtime setup
└── package.json
```

---

## Pages

| Route | Purpose |
|-------|---------|
| `/` | Event listings |
| `/seats?eventId=` | Seat map and reservation |
| `/checkout?seatIds=` | Timer + simulated payment |
| `/success` | Booking confirmation |
| `/my-tickets` | Past bookings and ticket download |
| `/login`, `/signup` | Authentication |
| `/support` | Contact / support form |

---

## Production notes

- Set **`REDIS_URL`** in production so locks work across multiple server instances.
- Restrict the Firebase **API key** in [Google Cloud Console](https://console.cloud.google.com/) (HTTP referrer limits).
- Do not commit `.env.local`. Use your host’s secret manager (Vercel, etc.) for env vars.
- Review Supabase **RLS policies** before going live — the included schema uses permissive demo policies.

---

## Troubleshooting

| Issue | What to check |
|-------|----------------|
| Seats not loading | `NEXT_PUBLIC_SUPABASE_*` in `.env.local`; run `supabase_schema.sql` |
| “Permission denied” on seats | RLS policies in schema; anon key matches project |
| Realtime not updating | Realtime enabled on `seats` (step 6 in schema); Supabase plan supports Realtime |
| Lock works locally but not in prod | Set `REDIS_URL`; mock Redis is per-process only |
| Auth errors | Firebase Email/Password enabled; all `NEXT_PUBLIC_FIREBASE_*` vars set |
| Session expires too fast | Increase `PX` value in `lock-seat/route.ts` |

---

## License

This project is licensed under the [MIT License](./LICENSE).

---

## Author

Built as a smart ticket booking system demonstrating concurrent seat reservation patterns with Redis and real-time UX with Supabase.
