# Setup Guide — Self-Hosting n8n with Docker

This guide walks you through running the full automation stack locally (or on a VPS) and wiring up every credential needed by the five workflows.

---

## Table of Contents

1. [Prerequisites](#prerequisites)
2. [Environment Variables](#environment-variables)
3. [Starting the Stack](#starting-the-stack)
4. [Importing Workflows into n8n](#importing-workflows-into-n8n)
5. [Configuring Credentials](#configuring-credentials)
   - [PostgreSQL](#postgresql)
   - [WhatsApp Cloud API](#whatsapp-cloud-api)
   - [Meta Ads (Graph API)](#meta-ads-graph-api)
   - [Cloudflare R2](#cloudflare-r2)
   - [Instagram Graph API](#instagram-graph-api)
   - [Facebook Graph API](#facebook-graph-api)
   - [LinkedIn API](#linkedin-api)
   - [Exotel](#exotel)
6. [Exposing Webhooks to the Internet](#exposing-webhooks-to-the-internet)
7. [Database Schema Initialisation](#database-schema-initialisation)
8. [Activating Workflows](#activating-workflows)
9. [Troubleshooting](#troubleshooting)

---

## Prerequisites

| Tool | Version | Install |
|------|---------|---------|
| Docker | 24+ | [docs.docker.com](https://docs.docker.com/get-docker/) |
| Docker Compose | v2 | bundled with Docker Desktop |
| `ngrok` or a VPS | — | for public webhook URLs |

---

## Environment Variables

Copy the example file and fill in your values:

```bash
cp .env.example .env
```

Open `.env` in any text editor. Each variable is documented inline. Never commit `.env` — it is already listed in `.gitignore`.

---

## Starting the Stack

```bash
# Start n8n + PostgreSQL in detached mode
docker compose up -d

# Tail logs
docker compose logs -f n8n

# Optional: also start pgAdmin GUI at http://localhost:5050
docker compose --profile debug up -d
```

Verify services are healthy:

```bash
docker compose ps
```

Expected output:
```
NAME            STATUS          PORTS
n8n_postgres    running         0.0.0.0:5433->5432/tcp
n8n_engine      running         0.0.0.0:5678->5678/tcp
```

Open **http://localhost:5678** and sign in with:
- Username: value of `N8N_BASIC_AUTH_USER` (default: `admin`)
- Password: value of `N8N_BASIC_AUTH_PASSWORD` (default: `changeme`)

---

## Importing Workflows into n8n

1. Open n8n → **Workflows** (left sidebar)
2. Click **⊕ Add Workflow** → **Import from File**
3. Select a `.json` from the `workflows/` directory
4. Click **Save**
5. Repeat for each of the five files

Alternatively, use the n8n CLI:
```bash
# Inside the container
docker exec -it n8n_engine n8n import:workflow --input=/home/node/.n8n/workflows/01-lead-capture-meta-ads.json
```

---

## Configuring Credentials

All credentials are created in n8n at **Settings → Credentials → ⊕ Add Credential**.

Nodes that need configuration are marked with `⚙ CONFIGURE` in their Notes field inside the workflow canvas.

---

### PostgreSQL

**Credential Type:** `Postgres`

| Field | Value |
|-------|-------|
| Host | `postgres` (Docker service name) or `localhost` if running n8n outside Docker |
| Port | `5432` |
| Database | `n8n_db` (or `POSTGRES_DB` from `.env`) |
| User | `n8n` (or `POSTGRES_USER`) |
| Password | `n8n_secret` (or `POSTGRES_PASSWORD`) |
| SSL | Off (for local) |

**Name this credential:** `Business DB — PostgreSQL`

---

### WhatsApp Cloud API

**Credential Type:** `HTTP Header Auth`

1. Go to **Meta for Developers** → your App → **WhatsApp** → **API Setup**
2. Generate a **Permanent Access Token** (requires a System User in Meta Business Manager)
3. Note your **Phone Number ID** — add it to `.env` as `WHATSAPP_PHONE_NUMBER_ID`

| Field | Value |
|-------|-------|
| Name | `Authorization` |
| Value | `Bearer <YOUR_PERMANENT_ACCESS_TOKEN>` |

**Name this credential:** `WhatsApp Cloud API`

> **Tip:** Temporary tokens expire after 24 hours. Always use a System User permanent token in production.

---

### Meta Ads (Graph API)

**Credential Type:** `HTTP Header Auth`

1. Go to **Meta Business Manager** → **System Users** → create a system user
2. Generate a **Long-Lived Token** with `ads_read`, `leads_retrieval`, `pages_read_engagement` permissions
3. Store the token in `.env` as `META_ADS_ACCESS_TOKEN`

| Field | Value |
|-------|-------|
| Name | `Authorization` |
| Value | `Bearer <LONG_LIVED_TOKEN>` |

**Name this credential:** `Meta Ads Access Token`

---

### Cloudflare R2

**Credential Type:** `S3`

1. Log in to [Cloudflare Dashboard](https://dash.cloudflare.com) → **R2** → **Manage API Tokens**
2. Create a token with **Object Read & Write** on your bucket
3. Note the **Account ID** from the R2 overview page

| Field | Value |
|-------|-------|
| Access Key ID | Your R2 token Access Key ID |
| Secret Access Key | Your R2 token Secret Access Key |
| Region | `auto` |
| Endpoint | `https://<ACCOUNT_ID>.r2.cloudflarestorage.com` |
| Force Path Style | `true` |

**Name this credential:** `Cloudflare R2`

> Create the bucket (`expense-receipts` or whatever you set as `R2_BUCKET_NAME`) before activating the workflow.

---

### Instagram Graph API

**Credential Type:** `HTTP Header Auth`

1. Your Facebook App must have **Instagram Graph API** product added
2. Connect your **Instagram Business** or **Creator** account to a Facebook Page
3. Generate a **Long-Lived Page Access Token** (60-day); set up a token refresh cron job for production

| Field | Value |
|-------|-------|
| Name | `Authorization` |
| Value | `Bearer <INSTAGRAM_LONG_LIVED_TOKEN>` |

**Name this credential:** `Instagram Graph API`

Add `INSTAGRAM_BUSINESS_ACCOUNT_ID` to `.env` (found in Meta Business Suite → Instagram account settings).

---

### Facebook Graph API

**Credential Type:** `HTTP Header Auth`

1. From your Facebook App → **Graph API Explorer**, generate a **Page Access Token** with `pages_manage_posts`, `pages_read_engagement`
2. Exchange for a long-lived token via `GET /oauth/access_token`

| Field | Value |
|-------|-------|
| Name | `Authorization` |
| Value | `Bearer <PAGE_ACCESS_TOKEN>` |

**Name this credential:** `Facebook Graph API`

Add `FACEBOOK_PAGE_ID` to `.env`.

---

### LinkedIn API

**Credential Type:** `HTTP Header Auth`

1. Create a LinkedIn App at [LinkedIn Developer Portal](https://www.linkedin.com/developers/apps)
2. Request the `w_member_social` and `r_organization_social` permissions
3. Complete OAuth 2.0 flow to get an access token (valid 60 days; use refresh tokens in production)

| Field | Value |
|-------|-------|
| Name | `Authorization` |
| Value | `Bearer <LINKEDIN_ACCESS_TOKEN>` |

**Name this credential:** `LinkedIn API`

Add `LINKEDIN_ORGANIZATION_ID` to `.env` (found in your LinkedIn Company Page URL: `linkedin.com/company/<ID>`).

---

### Exotel

Exotel does not require an n8n credential — it **calls** your webhook. What you need to configure is inside the Exotel dashboard:

1. Log in to [my.exotel.com](https://my.exotel.com)
2. Go to **Apps** → open your app (or create a Passthru app)
3. Under **Status Callback URL**, enter: `https://<n8n-host>/webhook/exotel-missed-call`
4. Save and verify the app is assigned to your Exotel number

---

## Exposing Webhooks to the Internet

For local development, use `ngrok` to create a public tunnel:

```bash
# Install ngrok: https://ngrok.com/download
ngrok http 5678
```

Note the HTTPS URL (e.g., `https://abc123.ngrok.io`) and:
1. Update `WEBHOOK_URL` in `.env` to this URL
2. Restart n8n: `docker compose restart n8n`
3. Use `https://abc123.ngrok.io/webhook/<path>` in Meta, Exotel, and your backend

**For production**, deploy n8n behind a reverse proxy (Nginx/Caddy) with a real domain and TLS. Set `WEBHOOK_URL` to your production domain.

---

## Database Schema Initialisation

Run the following SQL against your PostgreSQL instance to create all required tables. Connect via:

```bash
# Via psql in the Docker container
docker exec -it n8n_postgres psql -U n8n -d n8n_db
```

```sql
-- ── Leads ─────────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS leads (
  id            SERIAL PRIMARY KEY,
  leadgen_id    TEXT UNIQUE NOT NULL,
  full_name     TEXT,
  phone         TEXT,
  email         TEXT,
  city          TEXT,
  ad_name       TEXT,
  campaign_name TEXT,
  form_id       TEXT,
  status        TEXT DEFAULT 'new',
  created_at    TIMESTAMPTZ DEFAULT NOW()
);

-- ── Employees ─────────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS employees (
  id            SERIAL PRIMARY KEY,
  name          TEXT NOT NULL,
  phone         TEXT,
  manager_phone TEXT,
  department    TEXT,
  active        BOOLEAN DEFAULT true
);

-- ── GPS Check-ins ─────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS check_ins (
  id             SERIAL PRIMARY KEY,
  employee_id    INT REFERENCES employees(id),
  check_in_time  TIMESTAMPTZ,
  check_out_time TIMESTAMPTZ,
  lat_in         DECIMAL(10,7),
  lng_in         DECIMAL(10,7),
  lat_out        DECIMAL(10,7),
  lng_out        DECIMAL(10,7),
  status         TEXT DEFAULT 'present'
);

-- ── Attendance Reports ────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS attendance_reports (
  id            SERIAL PRIMARY KEY,
  report_date   DATE NOT NULL,
  total_present INT,
  total_absent  INT,
  total_late    INT,
  report_json   JSONB,
  sent_at       TIMESTAMPTZ DEFAULT NOW()
);

-- ── Expense Submissions ───────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS expense_submissions (
  id            SERIAL PRIMARY KEY,
  employee_id   INT REFERENCES employees(id),
  employee_name TEXT,
  phone         TEXT,
  caption       TEXT,
  amount        DECIMAL(10,2),
  currency      TEXT DEFAULT 'INR',
  receipt_url   TEXT,
  r2_key        TEXT,
  wa_msg_id     TEXT,
  submitted_at  TIMESTAMPTZ,
  status        TEXT DEFAULT 'pending_review'
);

-- ── Social Posts ──────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS social_posts (
  id            SERIAL PRIMARY KEY,
  post_id       TEXT UNIQUE,
  caption       TEXT,
  image_url     TEXT,
  platforms     TEXT[],
  ig_post_id    TEXT,
  fb_post_id    TEXT,
  li_post_id    TEXT,
  published_at  TIMESTAMPTZ DEFAULT NOW(),
  status        TEXT DEFAULT 'published'
);

-- ── Missed Calls ──────────────────────────────────────────────────────────
CREATE TABLE IF NOT EXISTS missed_calls (
  id            SERIAL PRIMARY KEY,
  call_sid      TEXT,
  caller_phone  TEXT,
  called_phone  TEXT,
  employee_id   INT REFERENCES employees(id),
  status        TEXT,
  started_at    TIMESTAMPTZ,
  wa_sent_at    TIMESTAMPTZ DEFAULT NOW()
);
```

---

## Activating Workflows

Once credentials are configured:

1. Open each workflow in n8n
2. Resolve any red error indicators on nodes (usually missing credentials)
3. Toggle the **Active** switch in the top-right corner
4. n8n will register the webhook URL and the workflow is live

---

## Troubleshooting

| Issue | Resolution |
|-------|-----------|
| Webhook returns `404` | Ensure the workflow is **Active** and the path matches |
| WhatsApp messages not sending | Check that the phone is in E.164 format and the token hasn't expired |
| R2 upload fails | Verify `Force Path Style = true` and the bucket exists |
| Meta webhook not firing | Re-subscribe the webhook field in Meta Developer Console |
| Postgres connection refused | Check `DB_POSTGRESDB_HOST=postgres` (the Docker service name, not `localhost`) |
| Cron not triggering | Confirm `GENERIC_TIMEZONE` is set correctly and the workflow is Active |
