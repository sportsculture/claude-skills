---
name: bootstrap-deploy
description: Provision a DigitalOcean droplet and deploy a Phoenix/Elixir app end-to-end using graybeam. Handles DNS, server setup (PostgreSQL, Caddy, asdf/Erlang), systemd service, graybeam config, build, deploy, and TLS verification.
---

# /bootstrap-deploy — Phoenix App Deployment to DigitalOcean

Automates the entire process of provisioning a DigitalOcean droplet and deploying a Phoenix/Elixir application using the graybeam deploy tool. Covers: droplet creation, DNS, server provisioning (PostgreSQL, Caddy, asdf, Erlang), systemd service, graybeam.toml configuration, build, deploy, and verification.

## Invocation

```
/bootstrap-deploy <app_name> <domain> [--port 4004] [--db-name app_prod] [--region nyc3] [--size s-1vcpu-1gb]
```

Examples:
- `/bootstrap-deploy brand_studio brand.spinculture.com --port 4004`
- `/bootstrap-deploy decision_forge forge.spinculture.com --port 4000 --db-name decision_forge_prod`
- `/bootstrap-deploy` -- Interactive mode, will ask for all parameters

## Prerequisites

- DigitalOcean API token (check project `.env` or `~/1-Projects/gridplay-v1/.env` for `DO_API_TOKEN`)
- SSH key ID `48160197` (spinculture-deploy-fedora) registered on DigitalOcean
- graybeam tool at `/home/mchughson/1-Projects/graybeam/graybeam`
- Phoenix project with `mix.exs` containing a `releases` config
- Local Elixir/Erlang installed (check `.tool-versions`)

---

# EXECUTION INSTRUCTIONS

## Step 1: Parse Arguments

Extract from `$ARGUMENTS`:
- `APP_NAME` — Elixir app name, snake_case (required)
- `DOMAIN` — Full domain, e.g. `brand.spinculture.com` (required)
- `PORT` — Phoenix port (default: `4004`)
- `DB_NAME` — PostgreSQL database name (default: `{APP_NAME}_prod`)
- `REGION` — DO region (default: `nyc3`)
- `SIZE` — Droplet size (default: `s-1vcpu-1gb`)

If `$ARGUMENTS` is empty or missing required params, use `AskUserQuestion`:
```
Question: "What is the app name (snake_case, e.g. brand_studio)?"
Header: "App Name"
```
```
Question: "What is the full domain (e.g. brand.spinculture.com)?"
Header: "Domain"
```

Derive:
- `APP_MODULE` — PascalCase of APP_NAME (e.g. `brand_studio` -> `BrandStudio`)
- `DB_USER` — Same as APP_NAME
- `DB_PASS` — Generate with: `openssl rand -base64 32 | tr -dc 'a-zA-Z0-9' | head -c 32`
- `SECRET_KEY_BASE` — Generate with: `mix phx.gen.secret` (run locally)
- `BASE_DOMAIN` — Parent domain extracted from DOMAIN (e.g. `spinculture.com` from `brand.spinculture.com`)
- `SUBDOMAIN` — Prefix extracted from DOMAIN (e.g. `brand` from `brand.spinculture.com`)

## Step 2: Collect API Keys for .env

Use `AskUserQuestion` to gather which API keys to include in the server `.env`:
```
Question: "Which API keys should be included in the production .env? List the ones you want (I'll ask for values). Common ones for this stack:
- VERTEX_API_KEY (Gemini image generation via Vertex AI)
- GOOGLE_API_KEY (Gemini fallback via AI Studio)
- ANTHROPIC_API_KEY (Claude API)
- TELEGRAM_BOT_TOKEN + TELEGRAM_BOT_USERNAME (Telegram auth)
- DO_SPACES_BUCKET, DO_SPACES_KEY, DO_SPACES_SECRET, DO_SPACES_REGION, DO_SPACES_CDN_URL (S3 storage)

Or say 'copy from local' to read them from the project's .env file."
Header: "Production API Keys"
```

If user says "copy from local", read the project `.env` and extract the relevant key-value pairs. Otherwise, ask for each key's value individually.

## Step 3: Locate DO API Token

```bash
# Check project .env first, then gridplay fallback
DO_TOKEN=$(grep -s 'DO_API_TOKEN=' .env ~/1-Projects/gridplay-v1/.env | head -1 | cut -d= -f2)
```

If not found, use `AskUserQuestion`:
```
Question: "I need a DigitalOcean API token. Where can I find it, or paste it here."
Header: "DO API Token"
```

## Step 4: Verify DNS Setup

Before creating a droplet, verify the base domain is managed by DigitalOcean:

```bash
# Check DO manages the domain's DNS
dig $BASE_DOMAIN NS +short
```

If DO nameservers (`ns1.digitalocean.com`, etc.) are not returned, STOP and warn the user:
```
WARNING: $BASE_DOMAIN does not appear to use DigitalOcean nameservers.
DNS records created via the DO API won't resolve. Either:
1. Move DNS to DigitalOcean first
2. Manually create the A record at your current DNS provider after droplet creation
```

Also verify the domain exists on the DO account:
```bash
curl -s -H "Authorization: Bearer $DO_TOKEN" \
  "https://api.digitalocean.com/v2/domains/$BASE_DOMAIN" | python3 -m json.tool
```

If the domain doesn't exist on DO, STOP and tell the user to add it first.

## Step 5: Provision Droplet

```bash
DROPLET_RESPONSE=$(curl -s -X POST "https://api.digitalocean.com/v2/droplets" \
  -H "Authorization: Bearer $DO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "'$APP_NAME'",
    "region": "'$REGION'",
    "size": "'$SIZE'",
    "image": "ubuntu-24-04-x64",
    "ssh_keys": [48160197],
    "tags": ["phoenix", "'$APP_NAME'"]
  }')

DROPLET_ID=$(echo "$DROPLET_RESPONSE" | python3 -c "import sys,json; print(json.load(sys.stdin)['droplet']['id'])")
echo "Droplet ID: $DROPLET_ID"
```

**Wait for droplet to become active and get its IP:**

```bash
# Poll until active (up to 120 seconds)
for i in $(seq 1 24); do
  DROPLET_INFO=$(curl -s -H "Authorization: Bearer $DO_TOKEN" \
    "https://api.digitalocean.com/v2/droplets/$DROPLET_ID")
  STATUS=$(echo "$DROPLET_INFO" | python3 -c "import sys,json; print(json.load(sys.stdin)['droplet']['status'])")
  if [ "$STATUS" = "active" ]; then
    DROPLET_IP=$(echo "$DROPLET_INFO" | python3 -c "
import sys,json
networks = json.load(sys.stdin)['droplet']['networks']['v4']
print(next(n['ip_address'] for n in networks if n['type']=='public'))")
    echo "Droplet active at: $DROPLET_IP"
    break
  fi
  echo "Status: $STATUS — waiting 5s..."
  sleep 5
done
```

If IP is not obtained after 120s, STOP and report the error.

## Step 6: Create DNS A Record

```bash
curl -s -X POST "https://api.digitalocean.com/v2/domains/$BASE_DOMAIN/records" \
  -H "Authorization: Bearer $DO_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "type": "A",
    "name": "'$SUBDOMAIN'",
    "data": "'$DROPLET_IP'",
    "ttl": 300
  }' | python3 -m json.tool
```

Verify propagation (may take a minute):
```bash
dig $DOMAIN A +short
```

## Step 7: Wait for SSH Availability

```bash
echo "Waiting for SSH on $DROPLET_IP..."
for i in $(seq 1 30); do
  if ssh -o ConnectTimeout=3 -o StrictHostKeyChecking=accept-new root@$DROPLET_IP echo "SSH ready" 2>/dev/null; then
    break
  fi
  sleep 5
done
```

## Step 8: Server Setup — Create deploy User

```bash
ssh root@$DROPLET_IP bash -s << 'SETUP_USER'
set -e

# Create deploy user
useradd -m -s /bin/bash deploy

# Copy SSH authorized keys from root to deploy
mkdir -p /home/deploy/.ssh
cp /root/.ssh/authorized_keys /home/deploy/.ssh/
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# Grant deploy sudo for systemctl (no password)
echo 'deploy ALL=(ALL) NOPASSWD: /usr/bin/systemctl' > /etc/sudoers.d/deploy
chmod 440 /etc/sudoers.d/deploy

echo "deploy user created with SSH + systemctl sudo"
SETUP_USER
```

## Step 9: Server Setup — Install PostgreSQL

```bash
ssh root@$DROPLET_IP bash -s << SETUP_PG
set -e
apt-get update -qq
apt-get install -y -qq postgresql postgresql-contrib

# Start and enable PostgreSQL
systemctl enable postgresql
systemctl start postgresql

# Create database user and database
sudo -u postgres psql -c "CREATE USER $DB_USER WITH PASSWORD '$DB_PASS';"
sudo -u postgres psql -c "CREATE DATABASE $DB_NAME OWNER $DB_USER;"
sudo -u postgres psql -c "GRANT ALL PRIVILEGES ON DATABASE $DB_NAME TO $DB_USER;"

echo "PostgreSQL ready: $DB_NAME owned by $DB_USER"
SETUP_PG
```

## Step 10: Server Setup — Install Caddy

```bash
ssh root@$DROPLET_IP bash -s << 'SETUP_CADDY'
set -e
apt-get install -y -qq debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | tee /etc/apt/sources.list.d/caddy-stable.list
apt-get update -qq
apt-get install -y -qq caddy
SETUP_CADDY
```

Write Caddyfile with reverse proxy:

```bash
ssh root@$DROPLET_IP bash -s << SETUP_CADDYFILE
cat > /etc/caddy/Caddyfile << 'EOF'
$DOMAIN {
    reverse_proxy 127.0.0.1:$PORT
}
EOF

systemctl enable caddy
systemctl restart caddy
echo "Caddy configured for $DOMAIN -> 127.0.0.1:$PORT"
SETUP_CADDYFILE
```

## Step 11: Server Setup — Install asdf + Erlang OTP 27

**CRITICAL: This is required because `include_erts: false` in the release config means the release does NOT bundle the Erlang runtime. The server must have a matching Erlang/OTP installed.**

```bash
ssh root@$DROPLET_IP bash -s << 'SETUP_ASDF_DEPS'
set -e
# Install Erlang build dependencies
apt-get install -y -qq build-essential autoconf m4 libncurses5-dev \
  libwxgtk3.2-dev libwxgtk-webview3.2-dev libgl1-mesa-dev libglu1-mesa-dev \
  libpng-dev libssh-dev unixodbc-dev xsltproc fop libxml2-utils libncurses-dev \
  openjdk-11-jdk libssl-dev
SETUP_ASDF_DEPS
```

```bash
# Install asdf and Erlang as deploy user
ssh root@$DROPLET_IP bash -s << 'SETUP_ASDF'
set -e
su - deploy << 'DEPLOY_CMDS'
set -e

# Install asdf
git clone https://github.com/asdf-vm/asdf.git ~/.asdf --branch v0.14.1
echo '. "$HOME/.asdf/asdf.sh"' >> ~/.bashrc
export PATH="$HOME/.asdf/bin:$HOME/.asdf/shims:$PATH"

# Install Erlang plugin and OTP 27
asdf plugin add erlang https://github.com/asdf-vm/asdf-erlang.git
echo "Installing Erlang OTP 27 — this takes 10-20 minutes on a small droplet..."
asdf install erlang 27.2
asdf global erlang 27.2

# Verify
erl -eval 'io:format("~s~n", [erlang:system_info(otp_release)]), halt().' -noshell
echo "asdf + Erlang OTP 27 installed for deploy user"
DEPLOY_CMDS
SETUP_ASDF
```

**WARNING:** Erlang compilation takes 10-20 minutes on an `s-1vcpu-1gb` droplet. Be patient. Consider using `--size s-2vcpu-2gb` to speed this up, then resizing down after.

## Step 12: Server Setup — App Directory Structure

```bash
ssh root@$DROPLET_IP bash -s << SETUP_DIRS
set -e
mkdir -p /var/www/$APP_NAME/shared
mkdir -p /var/www/$APP_NAME/releases
chown -R deploy:deploy /var/www/$APP_NAME
echo "Directory structure created at /var/www/$APP_NAME"
SETUP_DIRS
```

## Step 13: Server Setup — Write Shared .env

Build the `.env` content from the values collected in Step 2:

```bash
ssh root@$DROPLET_IP bash -s << SETUP_ENV
cat > /var/www/$APP_NAME/shared/.env << 'EOF'
DATABASE_URL=ecto://$DB_USER:$DB_PASS@localhost/$DB_NAME
SECRET_KEY_BASE=$SECRET_KEY_BASE
PHX_SERVER=true
PHX_HOST=$DOMAIN
PORT=$PORT
# API keys collected from user go here — one per line
# VERTEX_API_KEY=...
# ANTHROPIC_API_KEY=...
# (etc.)
EOF
chown deploy:deploy /var/www/$APP_NAME/shared/.env
chmod 600 /var/www/$APP_NAME/shared/.env
echo ".env written to /var/www/$APP_NAME/shared/.env"
SETUP_ENV
```

Replace the placeholder comment block with the actual API keys collected in Step 2.

## Step 14: Server Setup — Create systemd Service

**CRITICAL: The asdf PATH must be included in the `Environment` directive so the Erlang runtime is found at startup.**

```bash
ssh root@$DROPLET_IP bash -s << SETUP_SYSTEMD
cat > /etc/systemd/system/$APP_NAME.service << EOF
[Unit]
Description=$APP_NAME Phoenix application
After=network.target postgresql.service

[Service]
Type=exec
User=deploy
Group=deploy
WorkingDirectory=/var/www/$APP_NAME/current/$APP_NAME
Environment=PATH=/home/deploy/.asdf/shims:/home/deploy/.asdf/bin:/usr/local/bin:/usr/bin:/bin
EnvironmentFile=/var/www/$APP_NAME/shared/.env
ExecStart=/var/www/$APP_NAME/current/$APP_NAME/bin/$APP_NAME start
ExecStop=/var/www/$APP_NAME/current/$APP_NAME/bin/$APP_NAME stop
Restart=on-failure
RestartSec=5
SyslogIdentifier=$APP_NAME

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable $APP_NAME
echo "systemd service created and enabled: $APP_NAME"
SETUP_SYSTEMD
```

**Key detail:** `WorkingDirectory` and `ExecStart` use `current/$APP_NAME/` because graybeam syncs the release directory (e.g. `_build/prod/rel/brand_studio/`) which creates a nested folder under graybeam's `current` symlink.

## Step 15: Configure graybeam.toml

Write or update `graybeam.toml` in the project root:

```toml
[app]
name = "$APP_NAME"
keep_releases = 5

[target]
host = "$DROPLET_IP"
user = "deploy"
path = "/var/www/$APP_NAME"

[build]
command = "MIX_ENV=prod mix deps.get --only prod && MIX_ENV=prod mix assets.deploy && MIX_ENV=prod mix release --overwrite"

[sync]
include = ["_build/prod/rel/$APP_NAME/"]
exclude = [".git/", "deps/", "_build/dev/", "_build/test/", "node_modules/"]

[shared]
paths = [".env"]

[release]
healthcheck = "curl -fsS http://127.0.0.1:$PORT/health"
timeout_seconds = 30

[hooks]
pre_activate = ["export PATH=/home/deploy/.asdf/shims:/home/deploy/.asdf/bin:$PATH && set -a && source /var/www/$APP_NAME/shared/.env && set +a && $APP_NAME/bin/$APP_NAME eval '$APP_MODULE.Release.migrate()'"]
post_activate = ["sudo systemctl restart $APP_NAME"]
```

**Key lessons encoded in this config:**
- `include_erts: false` in mix.exs releases is **MANDATORY** (OpenSSL version mismatch between build machine and server otherwise)
- Hooks run via raw SSH so they need explicit `export PATH=...` for asdf shims
- Hooks need `set -a && source .env && set +a` to load environment variables
- The sync of `_build/prod/rel/{app}/` creates a nested directory, so paths in hooks reference `{app}/bin/{app}` (relative to the release directory)

## Step 16: Ensure mix.exs Has include_erts: false

Check the project's `mix.exs` for a releases config:

```bash
grep -A 5 'defp releases' mix.exs
```

If `include_erts: false` is NOT present, add it:

```elixir
defp releases do
  [
    APP_NAME: [
      include_erts: false
    ]
  ]
end
```

If there is no `releases` function at all, add it to the project and add `releases: releases()` to the `project/0` keyword list.

**Why:** Building on a dev machine (e.g. Fedora with OpenSSL 3.x) and deploying to Ubuntu produces OpenSSL version mismatches if ERTS is bundled. `include_erts: false` uses the server's Erlang (installed via asdf in Step 11).

## Step 17: Ensure Health Endpoint Exists

Check if the app has a `/health` endpoint:

```bash
grep -r '"/health"' lib/
```

If not, add a minimal health check route. In the router:

```elixir
get "/health", PageController, :health
```

And in the controller:

```elixir
def health(conn, _params) do
  conn |> put_resp_content_type("text/plain") |> send_resp(200, "ok")
end
```

## Step 18: Build Locally

```bash
MIX_ENV=prod mix deps.get --only prod
MIX_ENV=prod mix compile
MIX_ENV=prod mix assets.deploy
MIX_ENV=prod mix release --overwrite
```

Verify the release was built:
```bash
ls -la _build/prod/rel/$APP_NAME/bin/$APP_NAME
```

## Step 19: Deploy with graybeam

```bash
/home/mchughson/1-Projects/graybeam/graybeam deploy --skip-build --force
```

Use `--skip-build` because we already built in Step 18. Use `--force` for the first deploy since there's no previous release.

Watch the output for:
- Sync completing successfully
- pre_activate hook running migrations
- post_activate hook restarting the systemd service
- Healthcheck passing

## Step 20: Verify Deployment

### Check the service is running:
```bash
ssh deploy@$DROPLET_IP "sudo systemctl status $APP_NAME"
```

### Check the health endpoint:
```bash
curl -fsS http://$DROPLET_IP:$PORT/health
```

### Check TLS (Caddy auto-provisions Let's Encrypt cert):
```bash
curl -fsS https://$DOMAIN/health
```

If TLS fails, check Caddy logs:
```bash
ssh root@$DROPLET_IP "journalctl -u caddy --since '5 minutes ago' --no-pager"
```

### Check app logs:
```bash
ssh deploy@$DROPLET_IP "journalctl -u $APP_NAME --since '5 minutes ago' --no-pager"
```

## Step 21: Report Summary

Print a summary:

```
DEPLOYMENT COMPLETE

  App:      $APP_NAME
  Domain:   https://$DOMAIN
  Droplet:  $DROPLET_IP ($SIZE in $REGION)
  Database: $DB_NAME on localhost
  Port:     $PORT (behind Caddy reverse proxy)
  Service:  systemctl status $APP_NAME

  SSH:      ssh deploy@$DROPLET_IP
  Logs:     ssh deploy@$DROPLET_IP "journalctl -u $APP_NAME -f"
  Redeploy: /home/mchughson/1-Projects/graybeam/graybeam deploy

  graybeam.toml: written/updated in project root
```

---

# ERROR HANDLING

## Droplet Creation Fails
- Check DO API token is valid: `curl -s -H "Authorization: Bearer $DO_TOKEN" "https://api.digitalocean.com/v2/account" | python3 -m json.tool`
- Check SSH key ID exists: `curl -s -H "Authorization: Bearer $DO_TOKEN" "https://api.digitalocean.com/v2/account/keys/48160197" | python3 -m json.tool`
- Report the full API error response

## SSH Connection Fails
- Wait longer (some droplets take 60-90s to be SSH-ready)
- Verify the SSH key: `ssh -i ~/.ssh/id_ed25519 root@$DROPLET_IP`
- Check DO console if all else fails

## Erlang Build Fails on Small Droplet
- Memory exhaustion is common on 1GB droplets during Erlang compilation
- Fix: create a swap file before building:
```bash
ssh root@$DROPLET_IP bash -s << 'SWAP'
fallocate -l 2G /swapfile
chmod 600 /swapfile
mkswap /swapfile
swapon /swapfile
echo '/swapfile none swap sw 0 0' >> /etc/fstab
SWAP
```
- Then retry the asdf install erlang step

## Graybeam Deploy Fails
- "Permission denied": Check deploy user owns `/var/www/$APP_NAME`
- "Healthcheck failed": Check app logs with `journalctl -u $APP_NAME`
- "Connection refused on port": Verify PORT in .env matches systemd config
- Hook failures: SSH into server and run the hook command manually to see the error

## Caddy TLS Fails
- DNS not propagated yet: wait 2-5 minutes, then `sudo systemctl restart caddy`
- Rate limited by Let's Encrypt: check `journalctl -u caddy`
- Firewall blocking port 80/443: `ufw allow 80 && ufw allow 443` (or check DO firewall)

## OpenSSL Mismatch at Runtime
- Symptom: `libcrypto.so.3: cannot open shared object file` or similar
- Cause: Release was built with `include_erts: true` (the default)
- Fix: Set `include_erts: false` in mix.exs releases config, rebuild, redeploy

## Nested Directory Issue
- Symptom: `bin/$APP_NAME: No such file or directory`
- Cause: graybeam syncs `_build/prod/rel/$APP_NAME/` which creates `/var/www/$APP_NAME/current/$APP_NAME/`
- Fix: Ensure systemd WorkingDirectory and ExecStart use `current/$APP_NAME/bin/$APP_NAME`, not `current/bin/$APP_NAME`

---

# SUBSEQUENT DEPLOYS

After the initial bootstrap, future deploys are simple:

```bash
# From the project directory:
MIX_ENV=prod mix deps.get --only prod && MIX_ENV=prod mix assets.deploy && MIX_ENV=prod mix release --overwrite
/home/mchughson/1-Projects/graybeam/graybeam deploy --skip-build
```

Or let graybeam handle the build:
```bash
/home/mchughson/1-Projects/graybeam/graybeam deploy
```

To update server environment variables:
```bash
ssh deploy@$DROPLET_IP "vim /var/www/$APP_NAME/shared/.env"
# Then redeploy or restart:
ssh deploy@$DROPLET_IP "sudo systemctl restart $APP_NAME"
```
