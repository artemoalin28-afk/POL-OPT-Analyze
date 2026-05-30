# PolyOpt — Polymarket Portfolio Analytics & Hedge Toolkit

**PolyOpt** is a self-hosted web app for **Polymarket traders**. It brings together your positions, live market context, correlation-style linkages, **CVaR hedge suggestions**, optional **order placement**, and **alerts** — all running on **your own machine or server**, not a third-party SaaS.

> **Disclaimer:** Analytics, correlations, and optimizer output are **model-based decision support**, not investment advice. Polymarket trading involves risk; verify execution, fees, and settlement on your own.

---

## Table of contents

1. [What you get as a trader](#what-you-get-as-a-trader)
2. [Setup guide (no coding experience required)](#setup-guide-no-coding-experience-required)
3. [First time in the app](#first-time-in-the-app)
4. [Polymarket & wallet configuration](#polymarket--wallet-configuration)
5. [Environment variables reference](#environment-variables-reference)
6. [How the app works (technical overview)](#how-the-app-works-technical-overview)
7. [Production deployment](#production-deployment)
8. [Pro vs standard](#pro-vs-standard)
9. [Troubleshooting](#troubleshooting)
10. [Commands reference](#commands-reference)

---

## What you get as a trader

| Area | What it does |
|------|----------------|
| **Dashboard** | Overview of your Polymarket-linked portfolio. |
| **Portfolio strategy** | Positions, **RUN ALGORITHM** (CVaR optimizer), hedge suggestions, scenario presets. |
| **Live markets** | Per-market cards with prices, exposure-style metrics, spread/depth. |
| **Market scan** | Browse active markets by category (crypto, sports, politics). |
| **Correlations** | Similarity matrix across markets; drill into pairs. |
| **Hedge map** | Graph view of market linkages and concentration. |
| **Alerts** | Price / exposure rules with optional browser notifications. |
| **Integrations** | System health, wallet status, PWA install. |

**App pages (after login):**

| Page | URL path |
|------|----------|
| Portfolios | `/` |
| Portfolio detail | `/portfolio/1` |
| Live markets | `/markets` |
| Market scan | `/market-scan` |
| Correlations | `/correlations` |
| Hedge map | `/hedge-map` |
| Alerts | `/alerts` |
| Integrations | `/integrations` |

---

## Setup guide (no coding experience required)

This section assumes you are setting up PolyOpt **on your own computer** for personal use. You will copy and paste commands into a terminal. On Windows, use **PowerShell** (right-click Start → **Terminal** or **PowerShell**).

### What you need installed first

Install these **before** cloning the project:

| Software | Why you need it | Where to get it |
|----------|-----------------|-----------------|
| **Git** | Download the project code | [git-scm.com/downloads](https://git-scm.com/downloads) |
| **Node.js 20+** | Runs the web app server | [nodejs.org](https://nodejs.org/) (LTS version) |
| **PostgreSQL** | Stores your login sessions and alert data | [postgresql.org/download](https://www.postgresql.org/download/) |
| **Python 3.12+** | Powers the hedge optimizer | [python.org](https://www.python.org/downloads/) |
| **uv** (recommended) | Installs Python packages automatically | [docs.astral.sh/uv/getting-started](https://docs.astral.sh/uv/getting-started/installation/) |

**Check that installs worked** (copy each line, press Enter):

```powershell
git --version
node --version
npm --version
psql --version
python --version
uv --version
```

You should see version numbers, not “command not found”.

---

### Step 1 — Download PolyOpt

Pick a folder where you keep projects (e.g. `Documents` or `Downloads`), then run:

```powershell
cd $HOME\Downloads
git clone <your-repo-url> PolyOpt
cd PolyOpt
```

Replace `<your-repo-url>` with the actual Git URL of this repository.

---

### Step 2 — Install the web app dependencies

Still inside the `PolyOpt` folder:

```powershell
npm install
```

This may take a few minutes. Wait until it finishes without errors.

---

### Step 3 — Create a PostgreSQL database

During PostgreSQL installation you chose a **password** for the `postgres` user. Remember it.

**Option A — using pgAdmin (easier for beginners)**

1. Open **pgAdmin** (installed with PostgreSQL).
2. Connect to your local server.
3. Right-click **Databases** → **Create** → **Database**.
4. Name it `poly_opt` → Save.

**Option B — using the command line**

```powershell
psql -U postgres -c "CREATE DATABASE poly_opt;"
```

**Your database connection string** will look like:

```text
postgresql://postgres:YOUR_PASSWORD@localhost:5432/poly_opt
```

Replace `YOUR_PASSWORD` with your actual postgres password.

---

### Step 4 — Configure environment variables

PolyOpt reads settings from a file named `.env` in the project folder.

1. Copy the template:

   ```powershell
   copy .env.local.example .env
   ```

2. Open `.env` in **Notepad** or **VS Code** and edit these lines:

   | Setting | What to put |
   |---------|-------------|
   | `DATABASE_URL` | Your connection string from Step 3 |
   | `SESSION_SECRET` | Any long random string (64+ characters). Example generator in PowerShell: `-join ((48..57)+(65..90)+(97..122) \| Get-Random -Count 64 \| ForEach-Object {[char]$_})` |
   | `PORT` | `5003` (or any free port; default in code is `5000`) |
   | `POLY_ADDRESS` | Your **public** Polymarket / Polygon wallet address (`0x…`) — see [Polymarket section](#polymarket--wallet-configuration) |
   | `VITE_WALLETCONNECT_PROJECT_ID` | From [Reown Cloud](https://cloud.reown.com) — free project ID for wallet connect |

3. Save the file.

> **Never commit `.env` to Git.** It is already listed in `.gitignore` and may contain secrets.

---

### Step 5 — Create database tables

From the `PolyOpt` folder:

```powershell
npm run db:push
```

You should see Drizzle apply the schema successfully. If you get a connection error, double-check `DATABASE_URL` in `.env`.

---

### Step 6 — Install the Python optimizer

The **RUN ALGORITHM** button needs Python packages (`numpy`, `cvxpy`, etc.).

From the `PolyOpt` folder:

```powershell
uv sync
```

This creates a `.venv` folder and installs everything listed in `pyproject.toml`. Your `.env` should already contain:

```env
OPTIMIZER_CMD=uv
OPTIMIZER_ARGS=run python server/optimize.py
```

**Verify the optimizer works:**

```powershell
echo '{"request":{"scenario":"base"},"portfolio":{"id":1},"positions":[],"correlations":[]}' | uv run python server/optimize.py
```

You should see JSON output starting with `{"trades":` — not a Python error about missing `numpy`.

---

### Step 7 — Start the app

```powershell
npm run dev
```

When you see:

```text
Server listening at http://127.0.0.1:5003
```

(or whatever `PORT` you set), open that address in **Chrome** or **Edge**.

**Keep this terminal window open** while you use the app. Closing it stops the server.

**If you see “address already in use”:** another copy is still running. On Windows:

```powershell
netstat -ano | findstr :5003
taskkill /PID <number_from_last_column> /F
```

Then run `npm run dev` again.

---

### Step 8 — Create your login

1. Open the app URL in your browser.
2. Click **Create Account**.
3. Choose a username and password (stored locally in your PostgreSQL database — not Polymarket).
4. Sign in.

You now have a working PolyOpt instance. Continue with [First time in the app](#first-time-in-the-app) and optional Polymarket trading setup below.

---

### Setup checklist

Use this to confirm everything is ready:

- [ ] `npm install` completed
- [ ] `.env` exists with `DATABASE_URL` and `SESSION_SECRET`
- [ ] `npm run db:push` succeeded
- [ ] `uv sync` completed
- [ ] `npm run dev` shows “Server listening at …”
- [ ] You can register and log in at `/login`
- [ ] **Integrations** page (`/integrations`) shows Backend **OK**
- [ ] Portfolio / markets show data after setting `POLY_ADDRESS` (optional but recommended)

---

## First time in the app

### 1. Connect your wallet (optional but needed to place orders)

1. Use the wallet connect control in the app header.
2. Choose **MetaMask** (or another supported wallet).
3. Approve the connection on **Polygon** network.

If the wallet modal does not appear, set `VITE_WALLETCONNECT_PROJECT_ID` in `.env`, then **restart** `npm run dev`. For production builds, set it **before** running `npm run build`.

### 2. View your Polymarket positions

Set `POLY_ADDRESS` in `.env` to the **public address** of the Polymarket account you want to analyze. Restart the server.

- **Dashboard** and **Live markets** pull positions from Polymarket’s public Data API.
- If empty, check **Integrations** → Polymarket Data reachability, and confirm the address has open positions on [polymarket.com](https://polymarket.com).

### 3. Run the hedge optimizer

1. Go to **Portfolio** (usually `/portfolio/1`).
2. Adjust budget, risk tolerance, and scenario preset if desired.
3. Click **RUN ALGORITHM**.
4. Review suggested hedges in the results table.

If you see “ModuleNotFoundError: numpy”, run `uv sync` again and confirm `OPTIMIZER_CMD=uv` in `.env`.

### 4. Place an order (advanced, optional)

Placing orders requires **both**:

- A **connected browser wallet** (signs the order), and  
- **Server CLOB credentials** in `.env` (`POLY_API_KEY`, `POLY_PASSPHRASE`, `POLY_SECRET`) — see next section.

Until those are set, analytics and **RUN ALGORITHM** still work; **PLACE** will show “Order relay not configured”.

---

## Polymarket & wallet configuration

PolyOpt uses Polymarket in **two separate ways**. This confuses many traders — here is the plain-language split:

```text
┌─────────────────────────────────────────────────────────────┐
│  READ PATH (portfolio, markets, optimizer)                  │
│  Needs: POLY_ADDRESS (public 0x address only)               │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│  TRADE PATH (PLACE button → server → Polymarket CLOB)       │
│  Needs: POLY_ADDRESS + POLY_API_KEY + POLY_PASSPHRASE       │
│         + POLY_SECRET                                       │
│  Plus: browser wallet connected to sign the order           │
└─────────────────────────────────────────────────────────────┘
```

### `POLY_ADDRESS` — your public wallet address

- Copy from MetaMask → Account details → copy address.
- Used to load positions and market context from Polymarket.
- **Safe to store in `.env`** — it is public on-chain information.

### CLOB API credentials — for placing orders

These are **not** your MetaMask private key and **not** the Polymarket **Builder** tab keys ([polymarket.com/settings?tab=builder](https://polymarket.com/settings?tab=builder)). Builder keys are for a different program; PolyOpt uses standard **CLOB L2** credentials documented at [Polymarket Authentication](https://docs.polymarket.com/api-reference/authentication).

| `.env` variable | What it is |
|-----------------|------------|
| `POLY_API_KEY` | API key UUID from Polymarket CLOB |
| `POLY_PASSPHRASE` | Passphrase paired with that API key |
| `POLY_SECRET` | Secret used to sign API requests (server generates `POLY_SIGNATURE` / `POLY_TIMESTAMP` automatically) |

#### Generate credentials with the included script

1. Add your wallet private key **temporarily** to `.env`:

   ```env
   POLYMARKET_PRIVATE_KEY=0xYOUR_PRIVATE_KEY
   ```

   Use the private key for the **same account** as `POLY_ADDRESS`. Export from MetaMask: Account → ⋮ → Account details → Show private key.

2. Run the derive script from the project folder:

   ```powershell
   npx tsx tools/derive-poly-key.ts
   ```

3. Copy the JSON output into `.env`:

   | Script output | `.env` key |
   |---------------|------------|
   | `apiKey` or `key` | `POLY_API_KEY` |
   | `secret` | `POLY_SECRET` |
   | `passphrase` | `POLY_PASSPHRASE` |

4. **Remove `POLYMARKET_PRIVATE_KEY` from `.env`** — it is only for this one-time step.
5. Restart the server (`npm run dev`).

### WalletConnect / Reown project ID

1. Sign up at [cloud.reown.com](https://cloud.reown.com).
2. Create a project.
3. Copy the **Project ID** into `.env`:

   ```env
   VITE_WALLETCONNECT_PROJECT_ID=your-project-id-here
   ```

4. Restart dev server (or rebuild for production).

---

## Environment variables reference

Copy from [`.env.local.example`](.env.local.example). Only set what you need.

### Required to start the app

| Variable | Required? | Description |
|----------|-----------|-------------|
| `DATABASE_URL` | **Yes** | PostgreSQL connection string |
| `SESSION_SECRET` | **Yes** in Pro + production | Random string for login sessions. Dev mode has a fallback, but set your own anyway. |

### Recommended for traders

| Variable | Required? | Description |
|----------|-----------|-------------|
| `POLY_ADDRESS` | Recommended | Public Polygon address for portfolio / market feeds |
| `VITE_WALLETCONNECT_PROJECT_ID` | Recommended | Wallet connect modal (Reown / WalletConnect) |
| `PORT` | Optional | Default `5000`; many setups use `5003` |
| `APP_TIER=pro` | Optional | Enables Pro features (DB-backed alerts, rate limits, risk report API) |

### Order placement only

| Variable | Required? | Description |
|----------|-----------|-------------|
| `POLY_API_KEY` | For PLACE | CLOB L2 API key |
| `POLY_PASSPHRASE` | For PLACE | CLOB L2 passphrase |
| `POLY_SECRET` | For PLACE | CLOB L2 secret (HMAC signing on server) |
| `POLYMARKET_PRIVATE_KEY` | **Never for runtime** | One-time only in `.env` to run `derive-poly-key.ts`; delete afterward |

### Optimizer

| Variable | Default if unset | Description |
|----------|------------------|-------------|
| `OPTIMIZER_CMD` | `uv` | Command to run Python |
| `OPTIMIZER_ARGS` | `run python server/optimize.py` | Arguments passed to optimizer command |

### Optional tuning (have sensible defaults in code)

| Variable | Default |
|----------|---------|
| `HOST` | `127.0.0.1` (Windows) / `0.0.0.0` (Linux) |
| `FEED_STALE_WARNING_MS` | `120000` |
| `API_RATE_LIMIT_WINDOW_MS` / `API_RATE_LIMIT_MAX` | `60000` / `300` |
| `AUTH_RATE_LIMIT_WINDOW_MS` / `AUTH_RATE_LIMIT_MAX` | `900000` / `40` |
| `POLYMARKET_*_TIMEOUT_MS` | `20000` |
| `DEMO_MODE` | off |

`NODE_ENV` is set automatically: `npm run dev` → development, `npm start` → production.

### Optional frontend (Supabase live overlay)

If unset, the app uses built-in WebSocket polling instead.

| Variable | Description |
|----------|-------------|
| `VITE_SUPABASE_URL` | Supabase project URL |
| `VITE_SUPABASE_ANON_KEY` | Supabase anon key |
| `VITE_SUPABASE_MARKETS_CHANNEL` | Default `polyopt:markets` |
| `VITE_SUPABASE_ALERTS_CHANNEL` | Default `polyopt:alerts` |

---

## How the app works (technical overview)

| Layer | Technology | Role |
|-------|------------|------|
| **Frontend** | React 18, Vite, Tailwind, Wouter | Dashboard UI, wallet (Reown AppKit + Wagmi on Polygon) |
| **Backend** | Node.js, Express 5, TypeScript | REST API, sessions, WebSocket market polling |
| **Database** | PostgreSQL, Drizzle ORM | Users, sessions, Pro alert persistence |
| **Optimizer** | Python 3.12+, `server/optimize.py` | CVaR hedge engine (spawned on demand) |
| **External** | Polymarket Data / Gamma / CLOB APIs | Positions, metadata, order books, order relay |

**Request flow for RUN ALGORITHM:**

1. Browser → `POST /api/portfolios/:id/optimization`
2. Server loads portfolio state from Polymarket feeds
3. Server spawns `uv run python server/optimize.py` (or your `OPTIMIZER_*` command)
4. JSON result returned to the UI

**Request flow for PLACE order:**

1. Browser wallet signs order (EIP-712)
2. Browser → `POST /api/polymarket/order`
3. Server attaches L2 headers from `.env` → forwards to `https://clob.polymarket.com/order`

**Project layout (key folders):**

```text
PolyOpt/
├── client/          React frontend
├── server/          Express API, optimize.py, Polymarket clients
├── shared/          Shared types and API route definitions
├── tools/           derive-poly-key.ts (CLOB credential helper)
├── migrations/      SQL migrations (Pro alerts)
├── .env             Your local secrets (not in Git)
├── .env.local.example   Template for .env
└── pyproject.toml   Python optimizer dependencies
```

---

## Production deployment

For a VPS or dedicated server (Ubuntu recommended). There is **no Dockerfile** in this repo.

### Summary

1. Install Node 20+, PostgreSQL, Python 3.12+, and `uv`.
2. Clone repo, `npm ci`, copy `.env.local.example` → `.env`, fill production values.
3. `npm run db:push` and `uv sync`.
4. Set `VITE_*` variables, then `npm run build`.
5. `npm start` (serves API + static files from `dist/`).
6. Put **Nginx** or **Caddy** in front with HTTPS.
7. Use **systemd** or **PM2** to keep the process running.

### Production `.env` essentials

1. **`DATABASE_URL`** — production Postgres URL  
2. **`SESSION_SECRET`** — strong random value (required for `APP_TIER=pro`)  
3. **`POLY_ADDRESS`** — wallet to analyze  
4. **`PORT` / `HOST`** — e.g. `PORT=5000`, `HOST=0.0.0.0` behind a reverse proxy  
5. **`VITE_WALLETCONNECT_PROJECT_ID`** — must be set **before** `npm run build`  
6. **`APP_TIER=pro`** — optional; enables Pro checks (non-default `SESSION_SECRET` enforced in production)

### Build and run

```bash
npm ci
uv sync
npm run build
npm start
```

### Smoke test

1. `curl https://your-domain/api/health` — expect `"ok": true` and Polymarket reachability flags.  
2. Register / login in the browser.  
3. Open **Integrations** — Backend OK, Polymarket flags green.  
4. Run **RUN ALGORITHM** once on a portfolio page.

### Process manager example (PM2)

```bash
npm install -g pm2
cd /opt/polyopt
pm2 start dist/index.cjs --name polyopt --interpreter node
pm2 save
```

Ensure `NODE_ENV=production` (set by `npm start`).

---

## Pro vs standard

| Feature | Standard | Pro (`APP_TIER=pro`) |
|---------|----------|----------------------|
| Login & analytics | ✓ | ✓ |
| Optimizer | ✓ | ✓ |
| Alerts storage | Browser `localStorage` | PostgreSQL `user_alert_state` |
| API rate limits | Off | Per-IP limits on auth + API |
| Security headers | Baseline | Enhanced in production |
| Risk report API | — | `GET /api/pro/risk-report` |
| Production boot | Normal | Requires real `SESSION_SECRET` (not dev default) |

---

## Troubleshooting

| Symptom | What to do |
|---------|------------|
| **`EADDRINUSE` / port in use** | Stop the old server (`Ctrl+C`) or `taskkill` the PID from `netstat -ano \| findstr :PORT` |
| **DB error on boot** | Check `DATABASE_URL`; confirm Postgres is running; run `npm run db:push` |
| **`ModuleNotFoundError: numpy`** | Run `uv sync`; set `OPTIMIZER_CMD=uv` and `OPTIMIZER_ARGS=run python server/optimize.py` |
| **Empty portfolio / markets** | Set correct `POLY_ADDRESS`; check **Integrations** → Polymarket Data reachable |
| **Wallet modal missing** | Set `VITE_WALLETCONNECT_PROJECT_ID`; restart dev server |
| **PLACE fails / relay not configured** | Set `POLY_API_KEY`, `POLY_PASSPHRASE`, `POLY_SECRET` via `derive-poly-key.ts` — not Builder tab keys |
| **403 on order placement** | CLOB creds must match `POLY_ADDRESS` wallet; re-derive keys for that account |
| **`derive-poly-key.ts` errors** | Use full filename `.ts`; set `POLYMARKET_PRIVATE_KEY` in `.env` temporarily |
| **`npm start` exits immediately** | Pro + production needs non-default `SESSION_SECRET` |
| **502 / blank page in production** | Run `npm run build` first; confirm `dist/public` exists |
| **`getPlugin is not a function`** | Pull latest `vite.config.ts`; uses CommonJS-compatible import for `fe-utils-core` |

**Health check:** open `/integrations` while logged in, or visit `/api/health`.

---

## Commands reference

| Command | Description |
|---------|-------------|
| `npm install` | Install Node dependencies (first-time setup) |
| `npm run dev` | Development server + hot reload at `http://127.0.0.1:PORT` |
| `npm run build` | Build client → `dist/public`, server → `dist/index.cjs` |
| `npm start` | Production server (`NODE_ENV=production`) |
| `npm run db:push` | Apply database schema to PostgreSQL |
| `npm run check` | TypeScript type check |
| `uv sync` | Install Python optimizer into `.venv` |
| `npx tsx tools/derive-poly-key.ts` | Generate Polymarket CLOB API credentials |

---

## License

MIT — see `package.json`. Polymarket is a third-party service; this project is not affiliated with Polymarket.

---

## Contributing

Issues and PRs welcome: clearer trader-facing docs, Docker/Compose templates, and hardened production defaults are especially valuable.
