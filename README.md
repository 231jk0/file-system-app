# File System App

Browser-based folder/file tree (names only — no file content). Create folders and files, browse, search by exact name or prefix (top 10 typeahead), and delete.

- **Client:** React (Vite) in `file-system-app-client`
- **API:** Express + TypeScript in `file-system-app-server`
- **Database:** PostgreSQL 16

API details, schema, and trade-offs: [file-system-app-server/README.md](file-system-app-server/README.md).


|                  | Local development                              | Docker Compose                                                     |
| ---------------- | ---------------------------------------------- | ------------------------------------------------------------------ |
| **Use when**     | Changing code (hot reload)                     | Trying the full HTTPS stack, or deploying                          |
| **Open**         | [http://localhost:5173](http://localhost:5173) | [https://localhost.prijedlog.com](https://localhost.prijedlog.com) |
| **Env files**    | Client + server `.env`                         | Root `.env` only                                                   |
| **From scratch** | [Local development](#local-development)        | [Docker Compose](#docker-compose)                                  |




## Prerequisites

- Node.js 22+
- npm
- Docker and Docker Compose

There are no committed `.env` files. Copy each `.env.example` as shown. The example values work as-is.

## Local development

API on `http://localhost:3000`, UI on `http://localhost:5173`. Postgres runs in Docker (`postgres-for-development.yml`) on `localhost:5432`.

### 1. Clone

```bash
git clone https://github.com/231jk0/file-system-app.git
cd file-system-app
git submodule update --init --recursive
```

Each submodule checks out the **commit recorded in this repo**, not the tip of `main`. Detached HEAD (`HEAD detached at …`) is expected and is what you want for a reproducible run.

If you will change the client or server, attach them to `main` afterward:

```bash
git -C file-system-app-client checkout main
git -C file-system-app-server checkout main
```

`branch = main` in `.gitmodules` is only used by `git submodule update --remote`. It does not make `update --init` check out that branch.

After a later `git pull`, run `git submodule update --init --recursive` again if the client or server submodule moved.

### 2. Env files and install

From the repo root:

```bash
cp file-system-app-server/.env.example file-system-app-server/.env
cp file-system-app-client/.env.example file-system-app-client/.env
npm install --prefix file-system-app-server
npm install --prefix file-system-app-client
```

Leave the defaults unless your Postgres credentials differ:

**Server** (`file-system-app-server/.env`):

```
PORT=3000
CLIENT_URL=http://localhost:5173
DATABASE_URL=postgresql://myuser:mypassword@localhost:5432/mydb
```

**Client** (`file-system-app-client/.env`):

```
VITE_SERVER_URL=http://localhost:3000/api/v1
```



### 3. Postgres, migrations, then the app

From the repo root, in **two terminals**:

```bash
npm run postgres:up
npm run postgres:migrate-up
npm run dev:server
```

```bash
npm run dev:client
```

`postgres:up` waits until the container is healthy. `dev:server` exits if that container is not running.

Open [http://localhost:5173](http://localhost:5173). Stop the Node processes with Ctrl+C. Postgres stays up until `npm run postgres:down`.

**After a later** `git pull`**:** re-run the two `npm install` commands, then the same Postgres / migrate / two-terminal steps. Skip copying `.env` files if they already exist.

## Docker Compose

The root `docker-compose.yml` is the full stack: Traefik (HTTP→HTTPS), Postgres (internal network only), a one-shot migrate job, the API, and nginx serving the built UI. You do **not** copy the client or server `.env` files — Compose injects `DATABASE_URL`, `CLIENT_URL`, and the client API URL (`/api/v1`) at build/run time.

Ports **80** and **443** must be free on the host.

### On this machine (no public DNS)

The example `.env` uses `localhost.prijedlog.com`, which resolves to `127.0.0.1`. That is enough to try the production stack locally.

From the repo root:

```bash
cp .env.example .env
npm run docker-compose
```

`npm run docker-compose` tears the stack down, rebuilds images from current source, and starts it again.

Open [https://localhost.prijedlog.com](https://localhost.prijedlog.com). Let's Encrypt cannot issue a certificate for that hostname, so the browser will warn about a self-signed certificate — continue anyway.

### On a public host

Edit `.env`. `APP_DOMAIN` and `ACME_EMAIL` must be real; Postgres values can stay as in the example or be changed together:

```
APP_DOMAIN=example.com
ACME_EMAIL=you@example.com

POSTGRES_DB=mydb
POSTGRES_USER=myuser
POSTGRES_PASSWORD=mypassword
```

Create an A (or AAAA) record for `APP_DOMAIN` pointing at this machine's public IP. Traefik obtains a certificate via TLS-ALPN-01 on port 443.

Then start the same way:

```bash
npm run docker-compose
```

Open `https://APP_DOMAIN` (the value you set in `.env`).

### Logs and stop

```bash
docker compose logs -f
docker compose down
```

Data lives in the `postgres_data` volume until you remove volumes (`docker compose down -v`).