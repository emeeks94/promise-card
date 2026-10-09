# Promise Card

The Lagos Christmas promise card, back again. Make a card, share it on WhatsApp,
and friends and family write what they promise you — money, a gift, or something
from your Temu / Jumia / Konga wishlist.

Live at **https://promisecard.store**.

## What's in here

| Path | What it is |
|---|---|
| `src/app/` | Pages: landing, sign-in, make a card (`/start`), dashboard (`/me`), public card (`/c/<name>`) |
| `src/app/actions.ts` | Everything that writes data: make card, write promise, wishlist, mark collected |
| `src/app/c/[handle]/card.png/` | Generates the card image for WhatsApp link previews and Status |
| `src/db/schema.ts` | Database tables (Drizzle ORM, PostgreSQL) |
| `drizzle/` | SQL migrations, generated from the schema |
| `scripts/migrate.mjs` | Applies migrations (runs before each deploy) |
| `Dockerfile` | Production container (Next.js standalone, non-root) |
| `infra/` | Terraform for AWS (coming next) |

## Rules built in

- Card owners must confirm they're **18+**. Writing a promise needs no account.
- Wishlist links are limited to Temu, Jumia and Konga; nothing else is fetched.
- Spam protection: hidden honeypot field, 10 promises per hour per visitor, 300 per card.
- One promise per wishlist item, so two people can't promise the same thing.
- No money handled, no delivery addresses stored.

## Run it

### Option A: Docker (recommended, same on your PC and on EC2)

Needs Docker Desktop (or Docker Engine on Linux).

```bash
cp .env.example .env
# Edit .env: set NEXTAUTH_SECRET to a long random string
#   (Linux/Mac: openssl rand -base64 32  |  or any 40+ random characters)
docker compose up --build
```

Open http://localhost:3000. The first build takes a few minutes.

To sign in, enter any email, then get the link from the logs:

```bash
docker compose logs app | grep magic_link
```

Stop with `Ctrl+C`. Data is kept in a Docker volume; `docker compose down -v` wipes it.

### Option B: On an EC2 instance

1. Launch an instance (Amazon Linux 2023 or Ubuntu, t3.small or t4g.small is enough).
   Security group: allow port 22 from your IP and port 3000 from anywhere (for testing).
2. Install Docker and Git, copy or clone this folder onto it, then:

```bash
cp .env.example .env
# In .env set:
#   NEXTAUTH_SECRET=<long random string>
#   NEXTAUTH_URL=http://<ec2-public-ip>:3000
docker compose up --build -d
docker compose logs -f app
```

`NEXTAUTH_URL` must match the address in your browser exactly, or sign-in fails.
This is for testing. For the real launch, the database moves to RDS, HTTPS goes
on a load balancer, and it's all in Terraform (`infra/`).

### Option C: Without Docker

Needs Node 22 and PostgreSQL 16 running locally.

```bash
cp .env.example .env          # set NEXTAUTH_SECRET; DATABASE_URL points at your Postgres
npm ci
npm run db:migrate
npm run dev                   # http://localhost:3000
```

Without email settings, sign-in links are printed in the terminal instead of emailed.

### Changing the database

Edit `src/db/schema.ts`, then run `npm run db:generate` to create a new SQL file in
`drizzle/`. It's applied automatically on the next `docker compose up` (or `npm run db:migrate`).

## Environment variables

| Name | Purpose |
|---|---|
| `DATABASE_URL` | Postgres connection string |
| `DATABASE_SSL` | `true` on AWS RDS |
| `NEXTAUTH_URL` | Public site URL, e.g. `https://promisecard.store` |
| `NEXTAUTH_SECRET` | Random secret (`openssl rand -base64 32`) |
| `GOOGLE_CLIENT_ID` / `GOOGLE_CLIENT_SECRET` | Google sign-in (optional) |
| `EMAIL_SERVER` | SES SMTP URL, e.g. `smtp://USER:PASS@email-smtp.us-east-1.amazonaws.com:587` |
| `EMAIL_FROM` | `Promise Card <hello@promisecard.store>` |

## Health checks

- `GET /api/health` — is the server up (used by the load balancer)
- `GET /api/health?deep=1` — also checks the database (used by the uptime alarm)

Logs are one JSON object per line (`level`, `msg`, plus fields) so CloudWatch
Logs Insights can query them, e.g. `filter msg = "pledge_created" | stats count() by bin(1h)`.
