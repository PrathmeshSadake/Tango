# Tango

A BookMyShow-style movie ticket booking app. Pick a city, browse and search movies by date, choose a showtime, select your seats, and they are held for **7 minutes** while you pay. Nobody else can book them in that window.

Built with a Go API gateway, Go microservices, React, MySQL, Redis and RustFS, with Clerk for auth and Dodo Payments for checkout. The whole stack starts with one `docker compose up`.

---

## Features

- **Browse and search** movies by city, date, language, genre and format
- **Movie details** with showtimes grouped by theater
- **Interactive seat map** with seat categories, live availability and auto-selection of adjacent seats
- **7-minute seat hold** backed by Redis TTLs, with a countdown on screen
- **Secure payments** via Dodo Payments hosted checkout, confirmed by webhook
- **No double booking**: atomic Redis holds plus a MySQL unique constraint per show and seat
- **Automatic refunds** if a late payment arrives after the seats were taken
- **My bookings** with QR-code tickets
- **Admin panel** to manage movies, posters, theaters, screen layouts and shows
- **Self-hosted**: runs on a single EC2 instance or any VPS with automatic HTTPS

## Architecture

```mermaid
flowchart TB
    browser["Browser<br/>React app"]
    clerk["Clerk<br/>sign-in, JWTs"]:::ext
    dodo["Dodo Payments<br/>checkout, webhooks"]:::ext

    subgraph host["One server: docker compose up"]
        caddy["Caddy<br/>HTTPS, static files, /media"]
        web["web<br/>React build on Nginx"]
        gw["API gateway (Go)<br/>auth, rate limits, routing"]:::main

        subgraph svcs["Go services"]
            catalog["catalog<br/>cities, theaters, movies, shows"]
            booking["booking<br/>seat holds, bookings"]
            payment["payment<br/>checkout, webhooks, refunds"]
            notify["notification<br/>emails"]
        end

        mysql[("MySQL<br/>one database per service")]
        redis[("Redis<br/>holds, rate limits, event streams")]
        rustfs[("RustFS<br/>posters")]
    end

    browser -- sign in --> clerk
    browser -- HTTPS --> caddy
    caddy --> web
    caddy -- "/api/*" --> gw
    caddy -- "/media/*" --> rustfs
    gw -- verify JWT --> clerk
    gw -- REST --> catalog
    gw -- REST --> booking
    gw -- REST --> payment
    booking -- gRPC --> catalog
    payment -- gRPC --> booking
    payment <-- "checkout, webhooks" --> dodo
    catalog --> mysql
    booking --> mysql
    payment --> mysql
    notify --> mysql
    booking --> redis
    payment -. events .-> redis
    redis -. events .-> booking
    redis -. events .-> notify
    catalog --> rustfs

    classDef ext stroke-dasharray: 5 5
    classDef main stroke-width:3px
```

Only ports 80 and 443 are public, and both go to Caddy. The browser talks to Caddy, Clerk's sign-in widget and Dodo's hosted checkout. The gateway is the only public API; catalog, booking, payment and notification stay on the internal network. Dashed boxes are external SaaS.

| Component | Owns | Talks to |
| --- | --- | --- |
| **Caddy** | HTTPS certificates, serving the React build and posters | gateway, web, RustFS |
| **API gateway** | The public API surface: authentication, rate limiting, routing | catalog, booking, payment (HTTP) |
| **catalog** | Cities, theaters, screens, seat layouts, movies, shows, prices, posters | MySQL `catalog` DB, RustFS |
| **booking** | Seat holds, bookings, the seat map, confirming seats | MySQL `booking` DB, Redis, catalog (gRPC), events |
| **payment** | Checkout sessions, Dodo webhooks, refunds | MySQL `payment` DB, Dodo, booking (gRPC), events |
| **notification** | Confirmation and refund emails | MySQL `notification` DB, events, email provider |

### Three rules that shape everything

1. **Each service owns its data.** Only the booking service reads or writes the `booking` database. Others ask booking (gRPC) or listen to its events. This is what lets services change and deploy independently.
2. **Synchronous calls only when you need an answer now; events for everything else.** "What does this show cost?" needs an answer now (gRPC). "A payment succeeded" is a fact others react to in their own time (an event).
3. **Assume everything fails and everything happens twice.** Networks drop, webhooks retry, containers restart. Every handler is idempotent (safe to run twice) and every external call has a timeout.

### Why microservices

For a small team, a well-structured monolith is usually the right first step. Tango uses microservices because:

- **Different load profiles.** On a blockbuster release, seat holds spike 100× while the catalog barely changes. Booking can scale on its own.
- **Different failure blast radius.** If the email provider is down, bookings must still work. A separate notification service that consumes events makes that automatic.
- **Different security sensitivity.** Only the payment service holds payment API keys.
- **The patterns.** Gateway, service-to-service auth, outbox, sagas and idempotent consumers are what larger systems run in production.

The services sit in **one repository** (a monorepo) and **one Go module**, but each builds into its own Docker image and runs as its own process.

### Booking flow

```mermaid
sequenceDiagram
    actor U as User
    participant G as Gateway
    participant B as Booking
    participant C as Catalog
    participant R as Redis
    participant P as Payment
    participant D as Dodo
    participant N as Notification

    U->>G: POST /api/v1/shows/42/holds {seats}
    G->>G: verify Clerk JWT, rate limit
    G->>B: forward + X-User-Id
    B->>C: gRPC GetShow(42) (seats, prices)
    B->>R: Lua: hold all seats for 7 min, or none
    B-->>U: booking PENDING, expiresAt
    U->>G: POST /api/v1/payments/checkout {booking_id}
    G->>P: forward
    P->>B: gRPC GetBookingForCheckout
    P->>D: create checkout session
    P-->>U: checkout_url
    U->>D: pays
    D->>G: webhook payment.succeeded
    G->>P: forward (public route)
    P->>P: verify signature, save + outbox (one transaction)
    P--)R: event payment.succeeded
    R--)B: consume
    B->>B: insert booking_seats (unique per seat), CONFIRMED + outbox
    B--)R: event booking.confirmed
    R--)N: consume
    N->>U: confirmation email
    U->>G: GET /api/v1/bookings/{id} (polling)
    G->>B: forward
    B-->>U: CONFIRMED + ticket
```

Each held seat is a Redis key with a 420-second TTL, set by a Lua script that claims every requested seat or none of them. When the TTL expires, Redis deletes the keys and the seats free up. Confirmation inserts rows into `booking_seats`, which has `UNIQUE (show_id, seat_id)`. If a late payment hits a seat someone else already took, the insert fails and payment refunds.

## Tech stack

| Layer | Technology |
| --- | --- |
| Frontend | React 19, Vite, TypeScript, Bun, Tailwind CSS, shadcn/ui, TanStack Query, React Router |
| Backend | Go 1.23+, chi, sqlc, golang-migrate, slog |
| Auth | Clerk (`@clerk/clerk-react`, `clerk-sdk-go/v2`) |
| Payments | Dodo Payments (checkout sessions + webhooks) |
| Database | MySQL 8.4 |
| Cache / holds | Redis 7 |
| Object storage | RustFS (S3-compatible) |
| Proxy / TLS | Caddy 2 |
| Runtime | Docker Compose |

## Project structure

```text
tango/
├── api/                        # Go backend
│   ├── cmd/server/main.go
│   ├── internal/
│   │   ├── http/               # router, handlers, middleware (auth, rate limit)
│   │   ├── booking/            # seat holds, booking service
│   │   ├── payments/           # Dodo client + webhook handler
│   │   ├── catalog/            # cities, theaters, movies, shows, search
│   │   ├── storage/            # RustFS (S3) client, presigned uploads
│   │   └── db/                 # sqlc-generated queries
│   ├── migrations/
│   └── Dockerfile
├── web/                        # React + Vite + TypeScript (Bun)
│   ├── src/
│   ├── nginx.conf
│   └── Dockerfile
├── deploy/Caddyfile
├── scripts/                    # seed.sql, init-bucket.sh, backup.sh
├── docker-compose.yml
├── docker-compose.override.yml # dev only: exposes DB, Redis, RustFS ports
└── .env.example
```

## Getting started

### Prerequisites

- [Docker](https://docs.docker.com/get-docker/) with Docker Compose
- A [Clerk](https://clerk.com) application (publishable + secret key)
- A [Dodo Payments](https://dodopayments.com) account in test mode (API key + webhook secret)
- For local development outside Docker: [Bun](https://bun.sh) 1.x and Go 1.23+

### 1. Clone and configure

```bash
git clone https://github.com/<your-username>/tango.git
cd tango
cp .env.example .env
```

Fill in `.env`:

```bash
DOMAIN=localhost
APP_BASE_URL=https://localhost

MYSQL_DATABASE=tango
MYSQL_USER=app
MYSQL_PASSWORD=change-me
MYSQL_ROOT_PASSWORD=change-me-too
DB_DSN=app:change-me@tcp(mysql:3306)/tango?parseTime=true&loc=UTC

REDIS_ADDR=redis:6379
REDIS_PASSWORD=change-me

S3_ENDPOINT=http://rustfs:9000
S3_ACCESS_KEY=change-me
S3_SECRET_KEY=change-me-long-secret
S3_BUCKET=posters
PUBLIC_MEDIA_URL=https://localhost/media

CLERK_PUBLISHABLE_KEY=pk_test_...
CLERK_SECRET_KEY=sk_test_...

DODO_PAYMENTS_API_KEY=...
DODO_PAYMENTS_WEBHOOK_KEY=...
DODO_PAYMENTS_ENVIRONMENT=test_mode

SEAT_HOLD_SECONDS=420
```

### 2. Start everything

```bash
docker compose up -d --build
```

Then initialise storage and seed data:

```bash
./scripts/init-bucket.sh                                          # creates the posters bucket + public-read policy
docker compose exec -T mysql mysql -uapp -p tango < scripts/seed.sql
```

Open **https://localhost**. Caddy serves a locally trusted certificate in development.

### 3. Set up Dodo Payments

1. In the Dodo dashboard, create a one-time product called **Movie ticket** with **pay-what-you-want enabled**. The API sends each booking's server-computed total as the `amount`.
2. Under **Developer → Webhooks**, add `https://<your-domain>/api/webhooks/dodo`.
3. For local testing, expose the API with a tunnel and register that URL as a test-mode webhook:

```bash
cloudflared tunnel --url http://localhost:8080
```

### Local development without Docker

```bash
# Frontend
cd web
bun install
bun run dev

# Backend (MySQL, Redis and RustFS still run in Docker via the override file)
cd api
go run ./cmd/server
```

## API overview

| Method | Path | Auth | Purpose |
| --- | --- | --- | --- |
| GET | `/api/health` | — | Health check |
| GET | `/api/cities` | — | List cities |
| GET | `/api/movies?city=&date=&q=&genre=&lang=` | — | Browse and search movies |
| GET | `/api/movies/{slug}` | — | Movie details |
| GET | `/api/movies/{slug}/shows?city=&date=` | — | Showtimes by theater |
| GET | `/api/shows/{id}/seats` | optional | Seat map with availability |
| POST | `/api/shows/{id}/holds` | user | Hold seats for 7 minutes |
| DELETE | `/api/bookings/{id}/hold` | user | Release held seats |
| POST | `/api/bookings/{id}/checkout` | user | Create Dodo checkout session |
| GET | `/api/bookings/{id}` | user | Booking status and ticket |
| GET | `/api/me/bookings` | user | Booking history |
| POST | `/api/webhooks/dodo` | signature | Payment events |
| POST | `/api/admin/uploads/presign` | admin | Presigned poster upload |

## Deployment (EC2 or any VPS)

1. Launch an Ubuntu 24.04 instance (t3.medium recommended, 30 GB disk) and attach an **Elastic IP**.
2. Open only ports **22** (your IP), **80** and **443** in the security group.
3. Install Docker: `curl -fsSL https://get.docker.com | sh`.
4. Point an `A` record for your domain (for example `tango.yourdomain.com`) at the Elastic IP.
5. Switch Clerk and Dodo to **production / live** keys and add the live webhook URL.
6. Clone the repo, create `.env` with `DOMAIN=tango.yourdomain.com` and strong random secrets, then:

```bash
docker compose up -d --build
```

Caddy obtains a Let's Encrypt certificate automatically on the first request.

**Updating:**

```bash
git pull && docker compose up -d --build
```
