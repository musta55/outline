# Outline — Student Setup Guide

This is a **course fork of [outline/outline](https://github.com/outline/outline)** — a team wiki and
knowledge base with **real-time collaborative editing** (TypeScript, React + MobX, Koa, PostgreSQL,
Redis, ProseMirror). Two people can edit the same document and see each other's cursors live; that
sync engine is the most interesting part of this codebase.

📚 **Official documentation:** [https://docs.getoutline.com/s/hosting/](https://docs.getoutline.com/s/hosting/)

Verified end to end on macOS (Apple Silicon) and written for POSIX shells — on Windows use Git Bash
or WSL.

---

## ⚠️ Read this first — two things that will otherwise cost you an hour

1. **There is no real email.** Sign-in is a magic link delivered to a fake local mailbox
   (**Mailpit**, [http://localhost:8027](http://localhost:8027)). If you type your address at the login page and then check
   Gmail, nothing will ever arrive and you'll think the app is broken. **The link is in Mailpit.**
2. **You cannot create the first account yourself.** Outline normally bootstraps its first team
   through Google/Slack/OIDC single sign-on, which this course setup does not configure. The seed
   script in step 4 creates the team and users for you. Until you run it, the login page has **no
   sign-in options at all**.

---

## 1. Prerequisites

| Tool                     | Version          | Notes                                                           |
| ------------------------ | ---------------- | --------------------------------------------------------------- |
| **Node.js**        | **22**     | `engines` allows 20.12+, 22, 24 < 24.17, or 26 < 26.3.1 — use 22. |
| **Yarn**           | **4.11.0** | Pinned via `packageManager`; run `corepack enable` once.     |
| **Docker Desktop** | any recent       | Only for PostgreSQL, Redis, and Mailpit. The app runs natively. |
| **Git**            | any recent       |                                                                 |

Clone under a path with **no spaces or apostrophes** (e.g. `~/dev/...`) — shell scripts in this
monorepo break otherwise.

**macOS:** if Docker Desktop won't start and reports `VZErrorDomain Code=1`, **disable Rosetta** in
its settings. Every image used here is ARM-native.

---

## 2. Start the services

```bash
git clone <your-team-fork-url>
cd outline
corepack enable

docker compose up -d postgres redis
docker run -d --name outline-mailpit \
  -p 127.0.0.1:1027:1025 -p 127.0.0.1:8027:8025 axllent/mailpit
```

The `docker run` above is a **one-time** create. Run it a second time (say, after a reboot) and
Docker refuses with `Conflict. The container name "/outline-mailpit" is already in use`. That is
not a failure — the container already exists and just needs starting:

```bash
docker start outline-mailpit
```

---

## 3. Configure

```bash
cp .env.sample .env
openssl rand -hex 32    # run twice — one value for each secret below
```

Set these in `.env` (the rest of the sample can stay as-is). Note that `SMTP_HOST`,
`SMTP_PORT`, `SMTP_SECURE` and `SMTP_DISABLE_STARTTLS` are **not in `.env.sample`** — the
sample only ships `SMTP_SERVICE`, `SMTP_USERNAME`, `SMTP_PASSWORD` and `SMTP_FROM_EMAIL`, so
add those four lines yourself rather than searching for them. Leave `SMTP_USERNAME` and
`SMTP_PASSWORD` empty; Mailpit accepts unauthenticated mail.

```ini
NODE_ENV=production
URL=http://localhost:3003
PORT=3003
FORCE_HTTPS=false

SECRET_KEY=<first openssl value>
UTILS_SECRET=<second openssl value>

DATABASE_URL=postgres://user:pass@127.0.0.1:5432/outline
PGSSLMODE=disable
REDIS_URL=redis://127.0.0.1:6379

FILE_STORAGE=local
FILE_STORAGE_LOCAL_ROOT_DIR=<absolute path to a writable folder you create>

# Mailpit — where your sign-in links appear
SMTP_HOST=127.0.0.1
SMTP_PORT=1027
SMTP_FROM_EMAIL=hello@example.com   # the SENDER address — never a login
SMTP_SECURE=false
SMTP_DISABLE_STARTTLS=true

RATE_LIMITER_ENABLED=false
ENABLE_UPDATES=false
```

Port **3003** avoids clashes with the other course projects (3000 cal.diy, 3001 Actual,
3002 Excalidraw, 3030 Gitea, 4000 Discourse).

---

## 4. Build, migrate, seed, run

```bash
mkdir -p <the FILE_STORAGE_LOCAL_ROOT_DIR path you chose>

yarn install --immutable
yarn build                                # ~15 s
yarn db:migrate                           # 296 migrations, ~5 s
node build/server/scripts/seed-demo.js    # creates the team, users and demo content
yarn start
```

The app is at **[http://localhost:3003](http://localhost:3003)**.

---

## 5. Sign in

The seed script creates three users, one per permission level. There are no passwords.

| Email                  | Role                                    |
| ---------------------- | --------------------------------------- |
| `admin@example.com`  | Admin                                   |
| `member@example.com` | Member                                  |
| `viewer@example.com` | Viewer — handy for testing permissions |

> **Use one of the three addresses above — nothing else.** `hello@example.com` from the `.env`
> is the *sender* of the mail, not an account, and a typo is just as bad. Outline answers an
> unknown address with the same "check your email" screen it shows a real one (it refuses to
> reveal which accounts exist), then sends nothing at all. Mailpit stays empty and the app
> looks broken when in fact the address simply has no user.

1. Open [http://localhost:3003](http://localhost:3003) and enter one of the addresses.
2. Open [http://localhost:8027](http://localhost:8027) (Mailpit).
3. Open the newest "Magic Sign-in Link" email and click the link inside.

> ⚠️ **The link expires after 10 minutes** and is tied to the machine that requested it. A stale
> link bounces you straight back to the login page with
> `?notice=auth-error&description=Expired%20token` in the URL bar — which looks exactly like the
> login silently failing. Don't save links or paste them into chat; request a fresh one and click it
> immediately. Once you're in, the session lasts about three months.

You should land in the **CS435 Demo Wiki** workspace: three collections (Engineering, Product,
Playground) and seven documents.

---

## 6. Try the real-time collaboration

Worth doing before you pick a task — it's the feature that makes this project interesting.

1. Sign in as `admin@example.com` in a normal window.
2. Sign in as `member@example.com` in a **private/incognito window** (a second normal window shares
   cookies and won't work).
3. Open **Playground → Collaboration test** in both.
4. Type in one. It appears in the other as you type, with the other user's cursor and avatar.

---

## 7. Run the tests

The tests need **Postgres running** and their **own database** (`outline-test`), separate from the
`outline` database the app uses — so running them never touches your seeded demo content. Start the
services first. If you skip that, the first command fails with
`ERROR: connect ECONNREFUSED 127.0.0.1:5432`, which looks like a broken checkout but just means
Docker isn't up.

```bash
docker compose up -d postgres redis       # skip if they're already running

NODE_ENV=test yarn sequelize db:drop      # safe to re-run; wipes any existing test DB
NODE_ENV=test yarn sequelize db:create
NODE_ENV=test yarn sequelize db:migrate

yarn test                                 # full suite, ~75 s
```

The three `sequelize` lines are a one-time setup — after that `yarn test` on its own is enough.
Re-run them when you pull new migrations, or any time you want a clean slate.

Expected baseline — everything passes:

```
Test Files  292 passed (292)
     Tests  3669 passed | 6 skipped (3675)
```

While you're actually working, run something narrower — the full suite is slow:

```bash
yarn test server/models/User.test.ts      # a single file — ~4 s, use this most of the time
yarn test:server                          # backend only
yarn test:app                             # frontend only
yarn test:shared                          # shared/ only
yarn test:watch                           # re-runs on save
```

The suite is **Vitest** (not Jest), and `TZ=UTC` is already built into the `test` script.

Two harmless warnings you can ignore: a `client.query()` deprecation from the `pg` driver, and
missing sourcemaps for `prosemirror-codemark`.

---

## 8. Gotchas

1. **Login page shows no sign-in options** — you skipped the seed script, or it ran against a
   different database. Re-run `node build/server/scripts/seed-demo.js`.
2. **The magic link never arrives** — it's in Mailpit ([http://localhost:8027](http://localhost:8027)), not your inbox.
3. **Mailpit is empty and no link was ever sent** — you almost certainly signed in with an address
   that has no account, such as `hello@example.com` (that's `SMTP_FROM_EMAIL`, the sender) or a
   mistyped one. An unknown address still shows "check your email" but sends nothing, by design.
   Only `admin@`, `member@` and `viewer@example.com` exist; confirm with:

   ```bash
   docker exec outline-postgres-1 psql -U user -d outline -c 'select email, role from users;'
   ```

4. **You click the link and land back on the login page** — it expired (10-minute limit). Check the
   URL bar for `notice=auth-error`. Request a new one and click it right away.
5. **Port conflicts** — this project uses 3003 (app), 5432 (Postgres), 6379 (Redis), 8027/1027
   (Mailpit). Find the culprit with `lsof -nP -iTCP:3003 -sTCP:LISTEN`, or
   `netstat -ano | findstr "3003"` on Windows.
6. **Don't run `make up` unless you want the dev server.** It needs **mkcert** for local HTTPS and
   serves the client from a separate Vite server on port **3001**. The production path above is one
   process on one port and needs neither.
7. **No hot reload on this path.** After changing server code run `yarn build:server`; after
   changing client code run `yarn build`. Then restart `yarn start`.
8. **Anything fails with `ECONNREFUSED 127.0.0.1:5432` or `:6379`** — the Docker containers aren't
   running. Nothing restarts itself after a reboot, so start them again:

   ```bash
   docker compose up -d postgres redis
   docker start outline-mailpit
   ```

---

## 9. Daily workflow

```bash
docker compose up -d postgres redis
docker start outline-mailpit
yarn start
```

Reset to a clean seeded state:

```bash
yarn db:reset
node build/server/scripts/seed-demo.js
```

---

## 10. Where things live

| Path                      | What's in it                                           |
| ------------------------- | ------------------------------------------------------ |
| `app/`                  | React client (MobX state management)                   |
| `server/`               | Koa API server, Sequelize models, background workers   |
| `server/collaboration/` | the real-time sync engine — **change carefully** |
| `shared/`               | types and editor code used by both sides               |
| `plugins/`              | auth providers, storage backends, integrations         |

Read `docs/ARCHITECTURE.md` first. Good starter work lives in documents, collections, sharing, and
permissions — **not** the sync engine, where conflict-resolution bugs are subtle and hard to spot.
