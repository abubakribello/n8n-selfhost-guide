# Self-Hosting n8n on Hetzner Cloud with Cloudflare Tunnel

A beginner-friendly guide to running [n8n](https://n8n.io) on a cheap ARM server, served securely over HTTPS via Cloudflare Tunnel — no open firewall ports required.

![n8n](https://img.shields.io/badge/n8n-latest-orange) ![Docker](https://img.shields.io/badge/Docker-29.x-blue) ![License](https://img.shields.io/badge/license-MIT-green)

---

## Table of Contents

1. [Why This Stack](#why-this-stack)
2. [Prerequisites](#prerequisites)
3. [Cost Breakdown](#cost-breakdown)
4. [Architecture Overview](#architecture-overview)
5. [Step 1 — Provision the Hetzner Server](#step-1--provision-the-hetzner-server)
6. [Step 2 — Install Docker](#step-2--install-docker)
7. [Step 3 — Configure n8n with Docker Compose](#step-3--configure-n8n-with-docker-compose)
8. [Step 4 — Set Up Cloudflare Tunnel](#step-4--set-up-cloudflare-tunnel)
9. [Step 5 — Verify and Access n8n](#step-5--verify-and-access-n8n)
10. [Updating n8n](#updating-n8n)
11. [Troubleshooting](#troubleshooting)
12. [Security Notes](#security-notes)
13. [License](#license)

---

## Why This Stack

- **Hetzner CAX11** is one of the cheapest ARM VPS options available (~$4.59/month), with better price-to-performance than most competitors
- **Cloudflare Tunnel** eliminates the need to open any inbound firewall ports — the server connects outward to Cloudflare, not the other way around
- **PostgreSQL 15** instead of SQLite handles concurrent workflow executions reliably and is easier to back up
- **Free HTTPS** is handled entirely by Cloudflare — no Certbot or Let's Encrypt configuration needed

---

## Prerequisites

- A [Hetzner Cloud](https://console.hetzner.cloud) account — credit card or PayPal required
- A domain name you control (any registrar)
- A [Cloudflare](https://cloudflare.com) account — free tier is sufficient
- Your domain's nameservers pointed to Cloudflare (Cloudflare walks you through this when you add a domain)
- A terminal with SSH access (macOS/Linux built-in; Windows users can use WSL or PowerShell)

> All server commands in this guide are run as `root`. If you use a non-root user, prefix commands with `sudo`.

---

## Cost Breakdown

| Resource | Spec | Cost |
|----------|------|------|
| Hetzner CAX11 | 2 ARM vCPUs, 4 GB RAM, 40 GB NVMe | ~$4.59/month |
| Cloudflare Tunnel | Free tier | $0.00 |
| Domain | Varies by registrar and TLD | ~$1–15/year |
| **Total** | | **~$4.59/month + domain** |

Prices are approximate. Check [hetzner.com/pricing](https://hetzner.com/pricing) for current rates.

---

## Architecture Overview

No inbound firewall ports need to be open on the server. The tunnel is entirely initiated outbound by `cloudflared`:

```
User's Browser
      |
      | HTTPS (port 443)
      v
Cloudflare Edge  <--- handles TLS, DDoS protection
      |
      | Encrypted outbound tunnel (cloudflared)
      v
Hetzner Server  <--- no open inbound ports required
      |
      | localhost:5678
      v
  n8n Container
      |
      v
PostgreSQL 15 Container
```

`cloudflared` runs as a systemd service and starts automatically on reboot.

---

## Step 1 — Provision the Hetzner Server

1. Log in to [Hetzner Cloud Console](https://console.hetzner.cloud) and create a new project
2. Click **Add Server**
3. Choose settings:
   - **Location**: pick the region closest to you or your users
   - **Image**: Ubuntu 24.04
   - **Type**: Shared vCPU → **ARM64** tab → **CAX11**
   - **SSH Keys**: add your public key (strongly recommended)
4. Name the server (e.g. `n8n-server`) and click **Create & Buy Now**
5. Wait ~30 seconds, then note the server's **public IP address**

**To get your local SSH public key:**
```bash
cat ~/.ssh/id_ed25519.pub
```

If you don't have one yet, generate it first:
```bash
ssh-keygen -t ed25519 -C "hetzner-n8n" -f ~/.ssh/id_ed25519 -N ""
cat ~/.ssh/id_ed25519.pub
```

**Connect to your server:**
```bash
ssh root@YOUR_SERVER_IP
```

---

## Step 2 — Install Docker

Install Docker using the official convenience script:

```bash
curl -fsSL https://get.docker.com | sh
```

Verify both Docker and the Compose plugin are installed:

```bash
docker --version
docker compose version
```

Expected output: Docker 29.x and Docker Compose v2.x.

> **Note**: Running `docker compose up` may print a warning about the `version:` field being obsolete. This is harmless — see [Troubleshooting](#version-field-warning-in-docker-compose).

---

## Step 3 — Configure n8n with Docker Compose

### 3a — Create the project directory

```bash
mkdir ~/n8n && cd ~/n8n
```

### 3b — Create the `.env` file

This keeps your secrets separate from `docker-compose.yml`.

First, generate a secure encryption key:
```bash
openssl rand -hex 16
```

Then create the file:
```bash
nano .env
```

Paste and fill in your values (use `.env.example` from this repo as a template):
```env
DOMAIN=n8n.YOUR_DOMAIN.com
POSTGRES_PASSWORD=your_strong_db_password
N8N_ENCRYPTION_KEY=paste_openssl_output_here
N8N_USER=admin
N8N_PASSWORD=your_strong_n8n_password
```

Replace `n8n.YOUR_DOMAIN.com` with the subdomain you want to use (e.g. `n8n.mysite.com`).

> ⚠️ **Back up `N8N_ENCRYPTION_KEY` somewhere safe** (e.g. a password manager). It encrypts all credentials stored in the database and cannot be recovered if lost.

### 3c — Create `docker-compose.yml`

```bash
nano docker-compose.yml
```

Paste the contents from [`docker-compose.yml`](./docker-compose.yml) in this repo. Change `GENERIC_TIMEZONE` to your timezone — find yours at [this list](https://en.wikipedia.org/wiki/List_of_tz_database_time_zones).

### 3d — Start the containers

```bash
docker compose up -d
```

Verify both are running:
```bash
docker compose ps
```

Both `n8n` and `postgres` should show status `Up (healthy)` or `Up`.

---

## Step 4 — Set Up Cloudflare Tunnel

### 4a — Add your domain to Cloudflare

If your domain is not yet on Cloudflare:

1. Log in to [Cloudflare dashboard](https://dash.cloudflare.com) → click **Add a site**
2. Enter your domain → select **Free plan**
3. Note the two nameservers Cloudflare provides (e.g. `ada.ns.cloudflare.com`)
4. Go to your domain registrar and update nameservers to the two Cloudflare values
5. Wait for Cloudflare to show your domain as **Active** (usually minutes, up to 48 hours)

### 4b — Install cloudflared (ARM64)

```bash
curl -L https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-arm64 \
  -o /usr/local/bin/cloudflared
chmod +x /usr/local/bin/cloudflared
cloudflared --version
```

> If you chose a non-ARM instance type, use `cloudflared-linux-amd64` instead.

### 4c — Authenticate with your Cloudflare account

```bash
cloudflared tunnel login
```

A URL will be printed. **Open it in your browser within 8 minutes** (it expires — see [Troubleshooting](#tunnel-login-url-expired)). Log in and select your domain to authorize.

### 4d — Create the tunnel

```bash
cloudflared tunnel create n8n-tunnel
```

Note the **tunnel ID** (a UUID like `xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx`) printed in the output.

### 4e — Write the tunnel config

```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

Paste the following, replacing both instances of `TUNNEL_ID` with your UUID and updating the hostname (use the template from [`cloudflared-config.yml`](./cloudflared-config.yml) in this repo):

```yaml
tunnel: TUNNEL_ID
credentials-file: /root/.cloudflared/TUNNEL_ID.json

ingress:
  - hostname: n8n.YOUR_DOMAIN.com
    service: http://localhost:5678
  - service: http_status:404
```

### 4f — Route DNS to the tunnel

```bash
cloudflared tunnel route dns n8n-tunnel n8n.YOUR_DOMAIN.com
```

This automatically creates a CNAME record in your Cloudflare DNS — no manual dashboard steps needed.

### 4g — Run cloudflared as a system service

```bash
cloudflared service install
systemctl enable cloudflared
systemctl start cloudflared
```

Check it is running:
```bash
systemctl status cloudflared
```

Expected: `Active: active (running)`.

---

## Step 5 — Verify and Access n8n

1. Open your browser and go to `https://n8n.YOUR_DOMAIN.com`
2. You should see the n8n login screen
3. Log in with the `N8N_USER` and `N8N_PASSWORD` values from your `.env` file

Server-side health checks:
```bash
# Both containers should show "Up"
docker compose ps

# Tunnel should show "active (running)"
systemctl status cloudflared

# n8n should respond locally (expected: 200)
curl -o /dev/null -s -w "%{http_code}" http://localhost:5678/healthz
```

---

## Updating n8n

To update to the latest n8n release:

```bash
cd ~/n8n
docker compose pull
docker compose up -d
```

`docker compose pull` fetches the latest images. `docker compose up -d` recreates only changed containers. Your workflows, credentials, and database are stored in Docker volumes and are not affected.

Check the [n8n release notes](https://docs.n8n.io/release-notes/) before major version upgrades, as breaking changes occasionally occur.

---

## Troubleshooting

### Chrome "Dangerous Site" Warning

**Symptom**: Chrome shows a red warning page on first visit.

**Cause**: Chrome's Safe Browsing flags newly active domains until they establish a reputation. This is a false positive — your server and connection are secure.

**Fix**: Click **Details** → **Visit this unsafe site**. The warning disappears on its own within 2–5 days. Firefox and Safari do not show this warning.

---

### DNS Conflict — Cannot Use Root Domain

**Symptom**: `cloudflared tunnel route dns` fails with an error about an existing A or CNAME record.

**Cause**: Root domains often have a pre-existing A record (e.g. pointing to a registrar parking page). Cloudflare does not allow a CNAME at the root alongside other records.

**Fix**: Use a subdomain such as `n8n.yourdomain.com`. Update the `hostname:` in `~/.cloudflared/config.yml` and `DOMAIN=` in your `.env`, then restart:
```bash
docker compose down && docker compose up -d
systemctl restart cloudflared
```

---

### Tunnel Login URL Expired

**Symptom**: The browser authorization page says the link is invalid or expired.

**Cause**: The login URL expires after approximately 8 minutes.

**Fix**: Run `cloudflared tunnel login` again and complete the browser step promptly.

---

### `version` Field Warning in Docker Compose

**Symptom**: `docker compose up` prints: `the attribute 'version' is obsolete, it will be ignored`.

**Cause**: Docker Compose v2 deprecated the top-level `version:` field.

**Fix**: This is not an error — your containers start normally. You can delete the `version:` line from `docker-compose.yml` to silence the warning.

---

### n8n Container Keeps Restarting

**Symptom**: `docker compose ps` shows n8n restarting repeatedly.

**Checks**:
```bash
# Read the error
docker compose logs n8n --tail=100

# Check postgres is healthy
docker compose logs postgres --tail=50
```

**Common causes**:
- `N8N_ENCRYPTION_KEY` is empty or malformed — must be exactly 32 hex characters (output of `openssl rand -hex 16`)
- `POSTGRES_PASSWORD` mismatch between the n8n and postgres entries in `.env`
- PostgreSQL not yet healthy when n8n starts — try `docker compose restart n8n` after postgres shows `(healthy)`

---

## Security Notes

- Set strict permissions on your secrets file: `chmod 600 ~/n8n/.env`
- **Back up `N8N_ENCRYPTION_KEY`** in a password manager — it encrypts all credentials in the database and cannot be recovered if lost
- The port binding `127.0.0.1:5678:5678` in `docker-compose.yml` ensures n8n is only reachable via the Cloudflare Tunnel — not directly on the server's public IP
- Cloudflare's free tier provides DDoS protection and hides your server's real IP address
- The Hetzner firewall can be configured to allow only port 22 (SSH) inbound — the tunnel requires no open inbound ports
- Consider enabling [Cloudflare Access](https://developers.cloudflare.com/cloudflare-one/applications/configure-apps/) (free for up to 50 users) for an additional authentication layer in front of your subdomain

---

## License

MIT — use freely, adapt as needed.
