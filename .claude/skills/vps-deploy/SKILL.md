---
name: vps-deploy
description: Provision, harden and deploy the floorplan-3d white-label designer on this VPS (Docker Compose + Caddy auto-HTTPS + Postgres + Redis + backups), then print the embed snippet for client websites. Modes - setup (first time), update (pull latest and redeploy), status (health check), backup (run a backup now).
argument-hint: "[setup|update|status|backup]"
disable-model-invocation: true
---

# VPS deploy: floorplan-3d

You are running **on the production VPS** in a Claude Code session. Your job is to turn this server into the host for the white-label 3D home designer, and to keep it updated. Work step by step, show the user what you're about to do, and verify each step before moving on.

Mode requested: `$ARGUMENTS` (if empty, ask: setup, update, status or backup).

## Golden rules (never break these)

1. **Never lock the user out of SSH.** Don't disable password or root login until the user has confirmed, *from a second terminal*, that key login works for the `deploy` user. Always run `sshd -t` before reloading sshd. Use `reload`, never `restart`. Always `ufw allow OpenSSH` **before** `ufw enable`.
2. **Never print, echo, `cat` or commit secrets** (API keys, DB passwords, tokens). Write them only to `/opt/floorplan/.env` with `chmod 600`. When showing the file, mask values (`sed 's/=.*/=****/'`).
3. **Don't ask the user to paste secrets into chat.** For an API key, tell them to run `sudo nano /opt/floorplan/.env` themselves, or use `read -s` in their own terminal. Never pass a secret as a command-line argument.
4. **Ask before anything destructive or hard to reverse:** deleting volumes, dropping databases, `docker system prune`, changing the SSH config, enabling the firewall, overwriting an existing `/opt/floorplan`.
5. **Be idempotent.** Check what already exists before creating it. Rerunning `setup` must be safe.
6. **AI provider:** the backend uses the **Claude API** (`ANTHROPIC_API_KEY` from console.anthropic.com, pay per use). Never configure it to use a personal Claude Pro/Max subscription login for serving end users. That isn't permitted, and the whole service goes down if that account is limited. DeepSeek (`DEEPSEEK_API_KEY`) is an optional secondary provider.

## Layout on the server

```
/opt/floorplan/
  src/                 # git clone of the repo
  web/                 # static files served to visitors (copied from src on each deploy)
  Caddyfile
  docker-compose.yml
  .env                 # secrets, chmod 600, owned by deploy
  backups/             # local backups (also push off-site if configured)
  scripts/backup.sh
```

---

## Mode: `setup`

### Step 0: Collect inputs (ask the user, one message)
- `APP_DOMAIN`: e.g. `app.example.com` (required for HTTPS). If they have no domain yet, continue on plain HTTP by IP and warn them that client sites on HTTPS **can't** embed an HTTP iframe (mixed content), so they need a domain before going live.
- `API_DOMAIN`: e.g. `api.example.com` (optional now, needed once the backend exists).
- `ACME_EMAIL`: for Let's Encrypt notices.
- `REPO_URL`: default `https://github.com/hussainezzi/floorplan-3d.git`, and branch (default `master`). If the repo is private, they need a read-only **deploy key** (see Step 4).
- Remind them that DNS **A records** for each domain must point to this server's public IP. Show the IP with `curl -4 -s https://ifconfig.me`.

### Step 1: Preflight (read-only)
Run and summarise:
```bash
cat /etc/os-release | head -3; uname -m; nproc; free -h; df -h /
whoami; id; sudo -n true && echo "sudo OK"
ss -tlnp | grep -E ':(22|80|443)\s' || true
command -v docker && docker --version; docker compose version 2>/dev/null
ls -la /opt/floorplan 2>/dev/null
```
Stop and tell the user if: the OS isn't Ubuntu/Debian (adapt commands), there's less than 2 GB RAM, ports 80/443 are taken by another web server (ask whether to stop it), or `/opt/floorplan` already exists (offer `update` mode instead).

### Step 2: Base packages and updates
```bash
sudo apt-get update && sudo DEBIAN_FRONTEND=noninteractive apt-get -y upgrade
sudo apt-get install -y ca-certificates curl git ufw fail2ban unattended-upgrades jq
sudo dpkg-reconfigure -f noninteractive unattended-upgrades
sudo timedatectl set-timezone UTC
```
If RAM is ≤ 4 GB and no swap exists, offer to create a 2 GB swapfile.

### Step 3: `deploy` user and SSH hardening (careful: see Golden rule 1)
1. Create the user if missing: `sudo adduser --disabled-password --gecos "" deploy && sudo usermod -aG sudo deploy`.
2. Ask the user for their **public** key (`~/.ssh/id_ed25519.pub` on their laptop, which is safe to share). Append it to `/home/deploy/.ssh/authorized_keys` (dir 700, file 600, owner deploy). If the current user already has keys in `~/.ssh/authorized_keys`, offer to copy them.
3. **Stop and ask** the user to open a NEW terminal and run `ssh deploy@<server-ip>`, then confirm it worked. Do not continue hardening until they confirm.
4. Only after confirmation, write `/etc/ssh/sshd_config.d/60-hardening.conf`:
   ```
   PasswordAuthentication no
   KbdInteractiveAuthentication no
   PermitRootLogin prohibit-password
   ```
   Then `sudo sshd -t && sudo systemctl reload ssh` (the service may be named `sshd`).
5. Firewall:
   ```bash
   sudo ufw allow OpenSSH && sudo ufw allow 80/tcp && sudo ufw allow 443/tcp && sudo ufw allow 443/udp
   sudo ufw --force enable && sudo ufw status verbose
   ```
6. `sudo systemctl enable --now fail2ban`.

If the SSH port isn't 22, use that port in the ufw rule instead of `OpenSSH`.

### Step 4: Docker
If Docker is missing, install it from Docker's official apt repository (not the distro `docker.io` package), then:
```bash
sudo usermod -aG docker deploy
docker --version && docker compose version
```
Docker publishes ports past ufw. That's why only Caddy publishes ports below; Postgres and Redis must **never** have a `ports:` entry.

### Step 5: Get the code
```bash
sudo mkdir -p /opt/floorplan/{web,backups,scripts} && sudo chown -R deploy:deploy /opt/floorplan
git clone --branch <branch> <REPO_URL> /opt/floorplan/src
```
For a private repo: generate `ssh-keygen -t ed25519 -f ~/.ssh/floorplan_deploy -N ""` as deploy, show **only the .pub**, ask the user to add it at GitHub → repo → Settings → Deploy keys (read-only), then clone via the SSH URL using `GIT_SSH_COMMAND="ssh -i ~/.ssh/floorplan_deploy"`.

### Step 6: Secrets file
Create `/opt/floorplan/.env` (chmod 600, owner deploy). Generate the DB password yourself without printing it:
```bash
umask 077
cat > /opt/floorplan/.env <<EOF
APP_DOMAIN=<APP_DOMAIN>
API_DOMAIN=<API_DOMAIN or empty>
ACME_EMAIL=<ACME_EMAIL>
POSTGRES_USER=floorplan
POSTGRES_DB=floorplan
POSTGRES_PASSWORD=$(openssl rand -hex 24)
REDIS_PASSWORD=$(openssl rand -hex 24)
ANTHROPIC_API_KEY=
DEEPSEEK_API_KEY=
AI_MONTHLY_BUDGET_USD=50
EOF
```
Then tell the user to fill in `ANTHROPIC_API_KEY` themselves (`nano /opt/floorplan/.env`). They should also set a monthly spend limit in the Anthropic Console, because the embed is public and anyone visiting a client site can trigger AI calls.

### Step 7: Caddyfile
Write `/opt/floorplan/Caddyfile`. Framing must be **allowed**, because client sites embed us in an iframe.
```
{
	email {$ACME_EMAIL}
}

{$APP_DOMAIN} {
	encode zstd gzip
	root * /srv/web
	file_server
	header {
		# allow embedding on client websites (tighten to tenant domains later)
		Content-Security-Policy "frame-ancestors *"
		-X-Frame-Options
		Referrer-Policy strict-origin-when-cross-origin
		X-Content-Type-Options nosniff
		Permissions-Policy "camera=(), microphone=(self), geolocation=()"
	}
	@html path / /index.html
	header @html Cache-Control "no-cache"
}
```
Only if `API_DOMAIN` is set **and** the backend service exists (Step 8), append:
```
{$API_DOMAIN} {
	encode zstd gzip
	reverse_proxy api:3000
}
```
If no domain was given, use `:80` as the site address instead of `{$APP_DOMAIN}`.

### Step 8: docker-compose.yml
Write `/opt/floorplan/docker-compose.yml`:
```yaml
services:
  caddy:
    image: caddy:2.8
    restart: unless-stopped
    ports: ["80:80", "443:443", "443:443/udp"]
    env_file: .env
    volumes:
      - ./Caddyfile:/etc/caddy/Caddyfile:ro
      - ./web:/srv/web:ro
      - caddy_data:/data
      - caddy_config:/config

  postgres:
    image: postgres:16
    restart: unless-stopped
    env_file: .env
    volumes: [pg_data:/var/lib/postgresql/data]
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 10s
      retries: 5

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    env_file: .env
    command: ["sh", "-c", "exec redis-server --requirepass \"$$REDIS_PASSWORD\" --appendonly yes"]
    volumes: [redis_data:/data]

  # Backend: enabled only when src/apps/api exists (run with --profile api)
  api:
    profiles: ["api"]
    build: ./src/apps/api
    restart: unless-stopped
    env_file: .env
    environment:
      DATABASE_URL: postgres://$${POSTGRES_USER}:$${POSTGRES_PASSWORD}@postgres:5432/$${POSTGRES_DB}
      REDIS_URL: redis://:$${REDIS_PASSWORD}@redis:6379
    depends_on:
      postgres: { condition: service_healthy }
      redis: { condition: service_started }

volumes: { caddy_data: {}, caddy_config: {}, pg_data: {}, redis_data: {} }
```
Note: `$${VAR}` inside `environment:` doesn't interpolate from `env_file`. If the api service is enabled, have the api build `DATABASE_URL`/`REDIS_URL` from the individual vars, or move those values into the shell environment. Check with `docker compose config` (mask output) before relying on it.

### Step 9: Publish the frontend and start
```bash
cd /opt/floorplan
rsync -a --delete --exclude '.git' --exclude '.claude' --exclude 'docs' src/ web/
# if src has a build step later (package.json with "build"), run it and rsync its dist/ instead
docker compose pull && docker compose up -d
[ -d src/apps/api ] && docker compose --profile api up -d --build
docker compose ps
```

### Step 10: Backups
Write `/opt/floorplan/scripts/backup.sh` (chmod 700):
```bash
#!/usr/bin/env bash
set -euo pipefail
cd /opt/floorplan
ts=$(date -u +%Y%m%dT%H%M%SZ)
docker compose exec -T postgres sh -c 'pg_dump -U "$POSTGRES_USER" "$POSTGRES_DB"' | gzip > "backups/db-$ts.sql.gz"
tar czf "backups/uploads-$ts.tgz" -C /opt/floorplan web 2>/dev/null || true
find backups -type f -mtime +14 -delete
# Off-site (recommended): if rclone is configured with remote "offsite", sync:
command -v rclone >/dev/null && rclone listremotes | grep -q '^offsite:' && rclone copy backups offsite:floorplan-backups || true
```
Add a cron entry for deploy: `17 3 * * * /opt/floorplan/scripts/backup.sh >> /opt/floorplan/backups/backup.log 2>&1`. Run it once now and confirm a `.sql.gz` file appears. Recommend configuring `rclone` to an off-site bucket (Backblaze B2, Cloudflare R2 or S3), because a backup on the same VPS doesn't survive losing that VPS.

### Step 11: Verify
```bash
curl -sSI https://$APP_DOMAIN | head -5        # expect 200, and the TLS cert issued
curl -sS https://$APP_DOMAIN | grep -c '<title>'
curl -sSI https://$APP_DOMAIN | grep -i content-security-policy
docker compose logs --tail=50 caddy
```
If TLS fails, check that DNS points here (`dig +short $APP_DOMAIN`) and that ports 80/443 are open. Let's Encrypt rate-limits failures, so fix DNS before retrying many times.

### Step 12: Hand-over message to the user
Print:
1. The live URL.
2. The **embed snippet** for client websites:
   ```html
   <iframe src="https://APP_DOMAIN/"
           style="width:100%;height:80vh;border:0;border-radius:12px"
           allow="fullscreen; microphone" loading="lazy"
           title="3D Home Designer"></iframe>
   ```
   Per-tenant links (`/t/<tenant>`) and branding come later, when the backend exists. Until then every client sees the same demo plan.
3. A checklist of what's still pending: the API key in `.env`, off-site backups, and the API backend (not built yet).
4. How to update later: open Claude Code in `/opt/floorplan/src` and run `/vps-deploy update`.

---

## Mode: `update`
```bash
cd /opt/floorplan/src && git fetch origin && git status -sb
git log --oneline HEAD..@{u}            # show the user what's incoming
```
Ask for confirmation, then:
```bash
/opt/floorplan/scripts/backup.sh        # always back up before updating
git pull --ff-only
cd /opt/floorplan && rsync -a --delete --exclude '.git' --exclude '.claude' --exclude 'docs' src/ web/
docker compose pull && docker compose up -d
[ -d src/apps/api ] && docker compose --profile api up -d --build
docker compose ps
```
Then run the Step 11 checks. If something breaks, roll back with `git -C /opt/floorplan/src checkout <previous-sha>` and re-run the rsync and compose steps.

## Mode: `status`
Read-only report:
```bash
cd /opt/floorplan && docker compose ps
docker compose logs --since 1h --tail=30 caddy
df -h /; free -h; uptime
ls -lt backups | head -5
sudo ufw status; sudo fail2ban-client status sshd 2>/dev/null | tail -3
curl -sSI https://$(grep ^APP_DOMAIN .env | cut -d= -f2) | head -1
echo | openssl s_client -servername "$(grep ^APP_DOMAIN .env | cut -d= -f2)" -connect "$(grep ^APP_DOMAIN .env | cut -d= -f2)":443 2>/dev/null | openssl x509 -noout -enddate
```
Summarise: services up or down, disk and memory pressure (warn above 80%), age of the latest backup (warn if over 26h), TLS expiry, and any errors in the logs.

## Mode: `backup`
Run `/opt/floorplan/scripts/backup.sh`, then list the newest files in `backups/` with sizes.
