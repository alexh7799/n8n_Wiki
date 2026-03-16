# n8n Production Setup with Docker Compose

[To Home](../../README.md) | [← Back to Basic Install](./Docker.md)

> ⚠️ **This guide is for production deployments – not for local testing.**  
> "Production" means: n8n runs on a real server, is reachable from the internet, and stores data reliably.  
> If you just want to try n8n on your own computer first, start with the [Basic Install Guide](./Docker.md).

---

## 🧩 What are we setting up – and why?

The basic install guide uses a minimal single-container setup. That is fine for testing, but not for running n8n reliably on a real server. This guide adds three important pieces:

| Component | What it does | Why we need it |
| --------- | ------------ | -------------- |
| **Traefik** | Sits in front of n8n and handles all incoming traffic | Provides automatic HTTPS (SSL), redirects HTTP → HTTPS, and routes requests to n8n. Without this, your connection would be unencrypted. |
| **PostgreSQL** | A proper database server | The default n8n setup uses SQLite – a simple file-based database that is not reliable under load. PostgreSQL handles many simultaneous operations safely. |
| **Redis** | A fast in-memory data store | When n8n runs in "queue mode", Redis manages the list of pending workflow executions. This prevents data loss if n8n restarts mid-execution. |

Think of it like this: **Traefik** is the front door with a lock, **PostgreSQL** is the filing cabinet, and **Redis** is the whiteboard where current tasks are tracked.

---

## ✅ What you need before starting

- A **Linux server** (Ubuntu 22.04 or Debian 12 recommended) with Docker installed → [Install Guide](./Docker.md)
- A **domain name** that points to your server's IP address (e.g. `n8n.your-domain.com`)
  - You set this up in your domain registrar's DNS settings – add an **A record** pointing to your server's IP
  - DNS changes can take up to 24 hours to take effect, but usually only a few minutes
- **Ports 80 and 443** must be open on your server's firewall
  - Port 80 = regular HTTP (Traefik uses this briefly to redirect to HTTPS)
  - Port 443 = HTTPS (all actual traffic goes here)

---

## 📁 Folder Structure

By the end of this guide, your project folder will look like this:

```Code
n8n-production/
├── docker-compose.yml     ← defines all services (Traefik, PostgreSQL, Redis, n8n)
├── .env                   ← your passwords and settings (never share this file!)
└── traefik/
    ├── traefik.yml        ← Traefik configuration (how it behaves)
    └── acme.json          ← where Traefik stores your SSL certificate (created manually)
```

---

## Step 1: Create the Project Folder

```bash
mkdir n8n-production
cd n8n-production
mkdir traefik
```

---

## Step 2: Create the `.env` File

The `.env` file is where all your **passwords and personal settings** live. The `docker-compose.yml` reads values from this file using `${VARIABLE_NAME}` placeholders, so your secrets never have to be written directly into the compose file.

> 🔒 **This file must never be shared or committed to Git.** Add `.env` to your `.gitignore`.

Create a file named `.env` and fill in your own values:

```env
# ── Your Domain ───────────────────────────────────────────────────────────────
# The domain where n8n will be reachable (replace with your actual domain)
DOMAIN=n8n.your-domain.com

# ── Let's Encrypt (free SSL certificates) ────────────────────────────────────
# Traefik uses this email to register your SSL certificate with Let's Encrypt.
# You will receive an email here if your certificate is about to expire.
ACME_EMAIL=your-email@example.com

# ── PostgreSQL Database ───────────────────────────────────────────────────────
# These credentials are used to create and access the database.
# You can choose any values here – just be consistent.
POSTGRES_DB=n8n
POSTGRES_USER=n8n_user
POSTGRES_PASSWORD=CHANGE_ME_strong_password_1

# ── n8n Settings ──────────────────────────────────────────────────────────────
# The encryption key protects sensitive data stored by n8n (e.g. API credentials
# in your workflows). If you lose this key, you lose access to that data.
# Generate one with: openssl rand -hex 24
N8N_ENCRYPTION_KEY=CHANGE_ME_run_openssl_rand_hex_24

# Login credentials for the n8n web interface
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=CHANGE_ME_strong_password_2

# ── Timezone ──────────────────────────────────────────────────────────────────
# Used by n8n for scheduled workflows (cron triggers).
# Find your timezone name at: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
GENERIC_TIMEZONE=Europe/Berlin
TZ=Europe/Berlin
```

**Generate a secure encryption key** by running this command and copying the output:

```bash
openssl rand -hex 24
```

---

## Step 3: Create the Traefik Configuration File

Traefik needs to know how to behave: which ports to listen on, how to get SSL certificates, and how to find your containers. All of this is defined in `traefik/traefik.yml`.

Create `traefik/traefik.yml` with the following content:

```yaml
# ── API & Dashboard ───────────────────────────────────────────────────────────
# The Traefik dashboard is a web UI to inspect your routes and services.
# In production we disable it completely – it is not needed and would be
# an unnecessary attack surface.
api:
  dashboard: false
  insecure: false

# ── Entry Points ──────────────────────────────────────────────────────────────
# Entry points are the "doors" through which traffic enters Traefik.
entryPoints:
  web:
    address: ":80"               # Listen on port 80 (HTTP)
    http:
      redirections:
        entryPoint:
          to: websecure          # Immediately redirect all HTTP traffic...
          scheme: https          # ...to HTTPS
          permanent: true        # Tell browsers to always use HTTPS in the future

  websecure:
    address: ":443"              # Listen on port 443 (HTTPS)
    http:
      tls:
        certResolver: letsencrypt  # Automatically use an SSL certificate

# ── Providers ─────────────────────────────────────────────────────────────────
# A "provider" tells Traefik where to find routing rules.
# We use Docker – Traefik reads labels from your containers to know
# which domain should point to which container.
providers:
  docker:
    exposedByDefault: false    # IMPORTANT: only expose containers that explicitly
                               # have the label traefik.enable=true – everything
                               # else stays hidden from the internet
    network: traefik-public    # Only look at containers on this network

# ── SSL Certificate Resolver ──────────────────────────────────────────────────
# Traefik automatically requests free SSL certificates from Let's Encrypt.
# The certificate is stored in acme.json.
certificatesResolvers:
  letsencrypt:
    acme:
      tlsChallenge: {}         # Prove domain ownership via the TLS port (443)
      storage: /acme.json      # Where to save the certificate on disk
      # Note: the email address is passed in via an environment variable
      #       in docker-compose.yml – see the Traefik service section

# ── Logging ───────────────────────────────────────────────────────────────────
# Only log warnings and errors – keeps logs clean in production
log:
  level: WARNING

# Enable access logs (records every request – useful for debugging)
accessLog: {}
```

---

## Step 4: Prepare the SSL Certificate Storage File

Traefik needs a file to store the SSL certificate it gets from Let's Encrypt. You have to create this file manually and give it strict permissions – otherwise Traefik will refuse to start (it protects the certificate file from being read by other users).

```bash
touch traefik/acme.json
chmod 600 traefik/acme.json
```

What does `chmod 600` mean? It sets the file permissions so that **only the file owner can read and write it**. No other user on the system can access it.

---

## Step 5: Create the `docker-compose.yml`

This is the main file that defines all four services and how they connect to each other. Read through the comments – every important decision is explained.

Create `docker-compose.yml`:

```yaml
version: "3.8"

# ── Networks ──────────────────────────────────────────────────────────────────
# Think of networks like separate rooms. Containers can only talk to each other
# if they are in the same room (network).
#
# traefik-public: Traefik and n8n are both here – this is how Traefik can
#                 forward incoming web traffic to n8n.
# n8n-internal:   n8n, PostgreSQL and Redis are here – the database and cache
#                 are intentionally NOT in traefik-public, so they can never
#                 be reached directly from the internet.
networks:
  traefik-public:
  n8n-internal:

# ── Volumes ───────────────────────────────────────────────────────────────────
# Volumes are persistent storage managed by Docker.
# Even if a container is deleted and recreated, the data survives.
volumes:
  postgres_data:   # stores the PostgreSQL database files
  redis_data:      # stores Redis snapshots (so the queue survives restarts)
  n8n_data:        # stores n8n settings, credentials, and workflow files

services:

  # ── Traefik ───────────────────────────────────────────────────────────────
  # The reverse proxy – it receives all traffic on ports 80 and 443 and
  # forwards it to the right container. It also handles SSL automatically.
  traefik:
    image: traefik:v3.0
    container_name: traefik
    restart: unless-stopped
    environment:
      # We pass the ACME email here so it doesn't have to be written
      # directly inside traefik.yml (keeps config files clean)
      - TRAEFIK_CERTIFICATESRESOLVERS_LETSENCRYPT_ACME_EMAIL=${ACME_EMAIL}
    ports:
      - "80:80"      # HTTP – only used to redirect to HTTPS
      - "443:443"    # HTTPS – all real traffic goes here
    volumes:
      # Read-only access to Docker so Traefik can detect containers
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      # Our configuration file – mounted read-only for security
      - "./traefik/traefik.yml:/traefik.yml:ro"
      # The SSL certificate storage file
      - "./traefik/acme.json:/acme.json"
    networks:
      - traefik-public
    labels:
      # traefik.enable=false means Traefik does NOT create any route for
      # itself – the dashboard is fully unreachable from the internet
      - "traefik.enable=false"

  # ── PostgreSQL ────────────────────────────────────────────────────────────
  # The database. Stores all workflows, executions, users, and credentials.
  # Only reachable from inside the n8n-internal network – not from the internet.
  postgres:
    image: postgres:16-alpine     # alpine = smaller, more secure image
    container_name: n8n_postgres
    restart: unless-stopped
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    volumes:
      - postgres_data:/var/lib/postgresql/data
    networks:
      - n8n-internal
    # Healthcheck: Docker will wait for PostgreSQL to be actually ready
    # before starting n8n. Without this, n8n might try to connect before
    # the database is accepting connections and crash on first start.
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
      interval: 10s     # check every 10 seconds
      timeout: 5s       # give up after 5 seconds per check
      retries: 5        # mark as unhealthy after 5 failed checks

  # ── Redis ─────────────────────────────────────────────────────────────────
  # The queue manager. When n8n runs in "queue mode", every workflow execution
  # is first written to Redis before being picked up. This means if n8n
  # restarts, no executions are lost – they just wait in the queue.
  # Only reachable from inside the n8n-internal network.
  redis:
    image: redis:7-alpine
    container_name: n8n_redis
    restart: unless-stopped
    # --save 60 1 means: create a snapshot every 60 seconds if at least
    # 1 key has changed – this is how data survives a container restart
    command: redis-server --save 60 1 --loglevel warning
    volumes:
      - redis_data:/data
    networks:
      - n8n-internal
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 5s
      retries: 5

  # ── n8n ───────────────────────────────────────────────────────────────────
  # The main application. It only starts after both PostgreSQL and Redis
  # are healthy (see depends_on below).
  n8n:
    image: docker.n8n.io/n8nio/n8n:latest
    container_name: n8n
    restart: unless-stopped
    # Wait for the database and queue to be ready before starting
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    environment:
      # ── Database connection ──────────────────────────────────────────────
      - DB_TYPE=postgresdb
      - DB_POSTGRESDB_HOST=postgres        # "postgres" = the container name above
      - DB_POSTGRESDB_PORT=5432
      - DB_POSTGRESDB_DATABASE=${POSTGRES_DB}
      - DB_POSTGRESDB_USER=${POSTGRES_USER}
      - DB_POSTGRESDB_PASSWORD=${POSTGRES_PASSWORD}

      # ── Queue mode (uses Redis) ──────────────────────────────────────────
      # "queue" mode means workflow executions are managed via Redis.
      # This is more reliable than the default "regular" mode.
      - EXECUTIONS_MODE=queue
      - QUEUE_BULL_REDIS_HOST=redis        # "redis" = the container name above
      - QUEUE_BULL_REDIS_PORT=6379

      # ── Public URL & protocol ────────────────────────────────────────────
      # n8n needs to know its own public address to build correct webhook URLs
      - N8N_HOST=${DOMAIN}
      - N8N_PORT=5678
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://${DOMAIN}/

      # ── Encryption ───────────────────────────────────────────────────────
      # All sensitive data saved in n8n (API keys, passwords in credentials)
      # is encrypted with this key. Keep it safe and backed up.
      - N8N_ENCRYPTION_KEY=${N8N_ENCRYPTION_KEY}

      # ── Login protection ─────────────────────────────────────────────────
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}

      # Secure cookies only work over HTTPS – this prevents cookies from
      # being sent over unencrypted connections
      - N8N_SECURE_COOKIE=true

      # ── Privacy ──────────────────────────────────────────────────────────
      # Disable sending usage statistics to n8n's servers
      - N8N_DIAGNOSTICS_ENABLED=false
      - N8N_VERSION_NOTIFICATIONS_ENABLED=false

      # ── Timezone ─────────────────────────────────────────────────────────
      - GENERIC_TIMEZONE=${GENERIC_TIMEZONE}
      - TZ=${TZ}

    volumes:
      - n8n_data:/home/node/.n8n

    # n8n needs to be in both networks:
    # - traefik-public so Traefik can forward traffic to it
    # - n8n-internal so it can reach PostgreSQL and Redis
    networks:
      - traefik-public
      - n8n-internal

    labels:
      # Tell Traefik to route traffic for our domain to this container
      - "traefik.enable=true"
      - "traefik.http.routers.n8n.rule=Host(`${DOMAIN}`)"
      - "traefik.http.routers.n8n.entrypoints=websecure"
      - "traefik.http.routers.n8n.tls.certresolver=letsencrypt"
      - "traefik.http.services.n8n.loadbalancer.server.port=5678"

      # Security headers – these are instructions Traefik adds to every
      # response telling the browser how to behave securely:
      #
      # HSTS (Strict-Transport-Security): tells the browser to ALWAYS use
      # HTTPS for this domain, even if the user types http:// manually.
      # 31536000 seconds = 1 year
      - "traefik.http.middlewares.n8n-headers.headers.stsSeconds=31536000"
      - "traefik.http.middlewares.n8n-headers.headers.stsIncludeSubdomains=true"
      - "traefik.http.middlewares.n8n-headers.headers.stsPreload=true"
      - "traefik.http.middlewares.n8n-headers.headers.forceSTSHeader=true"
      # Prevent browsers from guessing file types (security best practice)
      - "traefik.http.middlewares.n8n-headers.headers.contentTypeNosniff=true"
      # Basic cross-site-scripting filter in older browsers
      - "traefik.http.middlewares.n8n-headers.headers.browserXssFilter=true"
      # Apply the headers middleware to the n8n router
      - "traefik.http.routers.n8n.middlewares=n8n-headers"
```

---

## Step 6: Set Up `.gitignore`

If you are using Git, make sure sensitive files are never accidentally committed:

```bash
cat <<EOF > .gitignore
.env
traefik/acme.json
EOF
```

---

## Step 7: Start Everything

```bash
docker compose up -d
```

**What happens now, step by step:**

1. Docker downloads all images (Traefik, PostgreSQL, Redis, n8n) – first time only, takes 1–3 minutes
2. PostgreSQL starts and initializes the database
3. Redis starts
4. Once both are healthy, n8n starts and connects to them
5. Traefik detects n8n via its Docker labels and requests an SSL certificate from Let's Encrypt
6. n8n is live at `https://n8n.your-domain.com`

Watch the startup in real time:

```bash
docker compose logs -f
```

Press `Ctrl + C` to stop watching the logs (this does **not** stop the containers).

Once you see a line like `n8n ready on 0.0.0.0, port 5678` in the logs, open your browser:

```Code
https://n8n.your-domain.com
```

You will be prompted to log in with the `N8N_BASIC_AUTH_USER` and `N8N_BASIC_AUTH_PASSWORD` from your `.env` file.

---

## ✅ Security Checklist

Go through this before sharing the URL with anyone:

- [ ] `.env` is listed in `.gitignore` and was never committed to a repository
- [ ] `traefik/acme.json` is listed in `.gitignore`
- [ ] `traefik/acme.json` has permission `600` (run `ls -la traefik/` to verify)
- [ ] `N8N_ENCRYPTION_KEY` was generated with `openssl rand -hex 24` – not typed by hand
- [ ] All passwords in `.env` are unique and not reused from other services
- [ ] Firewall: only ports **22 (SSH), 80 (HTTP), and 443 (HTTPS)** are publicly open
- [ ] n8n login is protected (`N8N_BASIC_AUTH_ACTIVE=true`)
- [ ] Traefik dashboard is disabled (`traefik.enable=false` on the Traefik container)
- [ ] Telemetry is off (`N8N_DIAGNOSTICS_ENABLED=false`)

---

## 🔧 Useful Commands

```bash
# Start all services in the background
docker compose up -d

# Stop all services (data is preserved)
docker compose down

# Watch live logs from all services
docker compose logs -f

# Watch logs from one service only
docker compose logs -f n8n
docker compose logs -f traefik
docker compose logs -f postgres

# Restart a single service (e.g. after changing .env)
docker compose restart n8n

# Update n8n to the latest version
docker compose pull n8n
docker compose up -d n8n

# Update all services at once
docker compose pull
docker compose up -d
```

> ⚠️ After updating n8n, always check the [n8n changelog](https://docs.n8n.io/release-notes/) for breaking changes before updating in production.

---

## 💾 Backup

**You should back up two things regularly:**

### 1. PostgreSQL Database (contains all workflows and execution history)

```bash
docker exec n8n_postgres pg_dump -U n8n_user n8n > backup_$(date +%Y%m%d_%H%M).sql
```

This creates a file like `backup_20260316_1430.sql` in your current folder.

### 2. n8n Data Volume (contains credentials and local files)

```bash
docker run --rm \
  -v n8n_data:/data \
  -v $(pwd):/backup \
  busybox tar czf /backup/n8n-data-$(date +%Y%m%d_%H%M).tar.gz /data
```

> 💡 **Tip:** Automate both backups with a cron job and copy the files to an offsite location (e.g. an S3-compatible storage like Backblaze B2 or Hetzner Object Storage).

---

## ♻️ Restore from Backup

### Restore PostgreSQL

```bash
# Stop n8n first so nothing writes to the DB during restore
docker compose stop n8n

# Restore the database dump
docker exec -i n8n_postgres psql -U n8n_user n8n < backup_20260316_1430.sql

# Start n8n again
docker compose start n8n
```

---

## 📚 Further Reading

- [n8n Hosting Documentation](https://docs.n8n.io/hosting/)
- [n8n Environment Variables](https://docs.n8n.io/hosting/configuration/environment-variables/)
- [n8n Queue Mode explained](https://docs.n8n.io/hosting/scaling/queue-mode/)
- [Traefik v3 Documentation](https://doc.traefik.io/traefik/)

---

Last updated: March 2026
