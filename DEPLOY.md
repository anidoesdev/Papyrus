# Deployment Guide — multi-project VM + Papyrus

Target: **`papyrus.anidoes.dev`** on VM **`162.43.26.54`**, a VM that hosts
several projects behind one shared nginx.

- **Part 0** — on your laptop (DNS, push, SSH)
- **Part 1** — one-time VM setup, shared by every project (Docker, firewall,
  swap, the common nginx files, catch-all, port registry)
- **Part 2** — deploy Papyrus
- **Part 3** — adding the next project
- Operations · Troubleshooting · Appendices

## How it fits together

```
                                   ┌─ bangkok-days.com         → (its own app)
browser ──▶ nginx :80/:443 ────────┼─ papyrus.anidoes.dev      → 127.0.0.1:3100 ─▶ papyrus frontend container
            one per VM, routes by  ├─ api.papyrus.anidoes.dev  → 127.0.0.1:8100 ─▶ papyrus api container ─┬─▶ db    127.0.0.1:5732
            domain (Host / SNI)    ├─ next-project.example.com → 127.0.0.1:3200 ─▶ …                        └─▶ redis 127.0.0.1:6380
                                   └─ anything else            → connection closed (catch-all)
```

- **One nginx owns ports 80/443** for the whole VM. It picks a site by the
  domain the browser asked for.
- **Each project** is its own directory in `/srv/apps/`, its own Docker Compose
  stack, its own nginx site file, its own certificate, and its own block of
  **localhost-only** ports.
- **Common nginx files**, installed once, hold the settings every site shares,
  so each project's site file stays short.

Every container port binds to `127.0.0.1`. Docker-published ports bypass
`ufw`, so that bind address is the only thing keeping them private — never
change it to `0.0.0.0`.

---

# Part 0 — On your laptop

## 0.1 Change DNS first (it takes up to an hour to spread)

At the DNS provider for `anidoes.dev`, set:

| Type | Name          | Value          |
|------|---------------|----------------|
| A    | `papyrus`     | `162.43.26.54` |
| A    | `api.papyrus` | `162.43.26.54` |

They currently point at `68.183.84.241` (an older server) with a 3600 s TTL,
so resolvers may keep the old answer for up to an hour. Do this now; it spreads
while you work through Part 1. Don't add AAAA (IPv6) records unless they point
at this VM — Let's Encrypt tries IPv6 first.

Check from PowerShell or any terminal:

```bash
nslookup papyrus.anidoes.dev 8.8.8.8
nslookup api.papyrus.anidoes.dev 8.8.8.8
```

## 0.2 Push the deployment files

The VM clones from GitHub, so the files this guide uses must be pushed first:

```bash
git add DEPLOY.md docker-compose.yml .env.example nginx/
git commit -m "Deploy behind shared host nginx"
git push origin main
```

## 0.3 SSH into the VM

Use whichever user and key you log in with; the examples assume `root` and
`~/.ssh/my_vm.txt`.

```bash
ssh -i ~/.ssh/my_vm.txt -o IdentitiesOnly=yes root@162.43.26.54
```

Optional shortcut — add to `~/.ssh/config` on your laptop, then just
`ssh papyrus-vm`:

```
Host papyrus-vm
    HostName 162.43.26.54
    User root
    IdentityFile ~/.ssh/my_vm.txt
    IdentitiesOnly yes
```

**Everything below runs on the VM.** Commands use `sudo`, so they work both as
`root` and as a sudo-capable user. Never run them from a service account such as
`postgres` — if `sudo` says `… is not in the sudoers file`, `exit` back to your
admin user.

---

# Part 1 — One-time VM setup (shared by every project)

Do this part once per VM. Later projects skip straight to their own Part 2.

## 1.1 Take stock before changing anything

Other sites already live here, so look first. These commands only read.

```bash
grep PRETTY_NAME /etc/os-release; nproc; free -h; df -h /
sudo ss -tlnp                                    # what is listening, and where
ls -l /etc/nginx/sites-enabled/
sudo grep -rn "server_name"    /etc/nginx/sites-enabled/
sudo grep -rn "default_server" /etc/nginx/sites-enabled/ /etc/nginx/conf.d/
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}' 2>/dev/null
command -v certbot && sudo certbot certificates
sudo ufw status verbose
```

Record how the existing sites answer right now — this is your "before" picture:

```bash
curl -sI https://bangkok-days.com | head -1      # repeat for every existing site
```

After each nginx change in this guide, run it again: the existing sites must
answer exactly the same.

## 1.2 Base packages

```bash
sudo apt update
sudo apt install -y curl git ca-certificates dnsutils openssl nginx
```

`nginx` is already installed here, so that part is a no-op. Only install
certbot if it's missing — if it came from snap, a second copy from apt causes
conflicts:

```bash
command -v certbot >/dev/null || sudo apt install -y certbot python3-certbot-nginx
sudo certbot plugins 2>/dev/null | grep -qi nginx && echo "certbot nginx plugin OK" \
  || sudo apt install -y python3-certbot-nginx
```

Optional: `sudo apt upgrade -y` for pending updates. On a live VM that can
briefly restart nginx or Docker (and with it every project's containers), so
pick a quiet moment.

## 1.3 Swap (protects builds from running out of memory)

A Next.js build can briefly need 1–2 GB on top of whatever is already running.
Without swap, the kernel kills the build (exit code 137). Add 2 GB if there is
no swap yet:

```bash
if ! swapon --show | grep -q .; then
  sudo fallocate -l 2G /swapfile
  sudo chmod 600 /swapfile
  sudo mkswap /swapfile
  sudo swapon /swapfile
  echo '/swapfile none swap sw 0 0' | sudo tee -a /etc/fstab
fi
free -h
```

## 1.4 Docker

```bash
command -v docker >/dev/null || curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker               # start now and on every boot
docker compose version
```

If you work as a non-root user, let it run Docker without `sudo` (the `docker`
group is effectively root, so only add admin users):

```bash
sudo usermod -aG docker $USER && newgrp docker
```

## 1.5 Firewall

Only SSH, 80 and 443 should be reachable from the internet. Check 1.1's output
first:

- **`ufw` is active** → make sure web traffic is allowed:
  ```bash
  sudo ufw allow 80/tcp
  sudo ufw allow 443/tcp
  sudo ufw status
  ```
- **`ufw` is inactive** → enabling it blocks every port without an allow rule.
  Allow SSH **first** (or you lock yourself out), plus every other public port
  1.1's `ss` output showed that must stay reachable:
  ```bash
  sudo ufw allow OpenSSH
  sudo ufw allow 80/tcp
  sudo ufw allow 443/tcp
  # sudo ufw allow <port>/tcp   ← for anything else that must stay public
  sudo ufw enable
  sudo ufw status
  ```

Many VPS providers also have a firewall in their control panel. If one is
enabled there, it must allow 22, 80 and 443 as well.

## 1.6 Common nginx files

Four files, installed once, reused by every project's site:

| File | Context | Purpose |
|---|---|---|
| `/etc/nginx/conf.d/00-common.conf` | `http` | Variables shared by all sites |
| `/etc/nginx/snippets/common-proxy.conf` | `location` | Standard reverse-proxy headers (incl. WebSockets) |
| `/etc/nginx/snippets/common-security.conf` | `server` | Security headers, hide nginx version |
| `/etc/nginx/sites-available/_template.example` | — | Starting point for each new project's site |

Make sure the variable name isn't already taken by an existing site:

```bash
sudo grep -rn 'ws_connection' /etc/nginx/ || echo "free to use"
```

**`00-common.conf`** — Ubuntu's `nginx.conf` includes `conf.d/*.conf` inside
the `http {}` block, so this loads automatically:

```bash
sudo tee /etc/nginx/conf.d/00-common.conf >/dev/null <<'EOF'
# Shared by every site on this VM (http context).

# Connection header for proxied requests: "upgrade" for WebSocket requests,
# "close" for everything else. Used by snippets/common-proxy.conf.
map $http_upgrade $ws_connection {
    default upgrade;
    ''      close;
}
EOF
```

**`common-proxy.conf`** — include it inside every `location` that uses
`proxy_pass`:

```bash
sudo tee /etc/nginx/snippets/common-proxy.conf >/dev/null <<'EOF'
# Standard reverse-proxy headers. Include inside each proxied location:
#     location / {
#         proxy_pass http://127.0.0.1:PORT;
#         include snippets/common-proxy.conf;
#     }
# Include it in the location, not the server block: a location that sets any
# proxy_set_header of its own silently drops every inherited one.
# Timeouts are deliberately not set here, so each site can set its own
# without a "duplicate directive" error.

proxy_http_version 1.1;

proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;

# WebSockets (Next.js, Socket.IO, etc.) — $ws_connection is in conf.d/00-common.conf
proxy_set_header Upgrade           $http_upgrade;
proxy_set_header Connection        $ws_connection;
EOF
```

**`common-security.conf`** — include it once per `server` block:

```bash
sudo tee /etc/nginx/snippets/common-security.conf >/dev/null <<'EOF'
# Baseline hardening. Include once per server block.
# Caveat: a location that uses add_header itself drops these headers for that
# location — re-include this snippet there if you ever add one.

server_tokens off;   # don't advertise the nginx version

add_header X-Content-Type-Options "nosniff"                         always;
add_header X-Frame-Options        "SAMEORIGIN"                      always;
add_header Referrer-Policy        "strict-origin-when-cross-origin" always;
EOF
```

**The template** — sits in `sites-available` without a symlink, so nginx never
loads it:

```bash
sudo tee /etc/nginx/sites-available/_template.example >/dev/null <<'EOF'
# Template for a new project's site. Not enabled — copy it:
#   sudo sed -e 's/APP_NAME/myapp/g' -e 's/APP_DOMAIN/myapp.example.com/g' \
#            -e 's/APP_PORT/3200/g' /etc/nginx/sites-available/_template.example \
#     | sudo tee /etc/nginx/sites-available/myapp >/dev/null
# Then symlink into sites-enabled, nginx -t, reload, and run certbot --nginx.

server {
    listen 80;
    listen [::]:80;
    server_name APP_DOMAIN;

    access_log /var/log/nginx/APP_NAME.access.log;
    error_log  /var/log/nginx/APP_NAME.error.log;

    include snippets/common-security.conf;

    location / {
        proxy_pass http://127.0.0.1:APP_PORT;
        include snippets/common-proxy.conf;
    }
}
EOF
```

Apply and confirm the existing sites are untouched:

```bash
sudo nginx -t && sudo systemctl reload nginx
curl -sI https://bangkok-days.com | head -1      # same as in 1.1
```

**Always `nginx -t` before `reload`** on a shared VM. A bad config aborts the
reload, and a stopped nginx takes every project down with it. Use `reload`, not
`restart`: it keeps the other sites' open connections alive.

## 1.7 Catch-all for unknown domains

When a request arrives for a domain nginx has no site for (someone browsing to
the bare IP, or a stranger's domain pointed at your server), nginx sends it to
the **default server**. Without one declared, that's whichever site loaded
first, so one of your real projects gets shown under someone else's name. A
catch-all closes those connections instead.

Only one block per port may be `default_server`. Check who holds it now:

```bash
sudo grep -rln "default_server" /etc/nginx/sites-enabled/
```

- **Nothing listed** → go straight to creating the catch-all below.
- **Only `default` listed** (Ubuntu's stock welcome page) → first confirm it
  serves no real domain:
  ```bash
  sudo grep -n "server_name" /etc/nginx/sites-enabled/default
  ```
  If it only shows `server_name _;`, disable it (the file stays in
  `sites-available`):
  ```bash
  sudo rm /etc/nginx/sites-enabled/default
  ```
- **A real site's file is listed**, or `default` has a real `server_name` →
  keep the site, just drop the flag from it (backup first):
  ```bash
  f=$(readlink -f /etc/nginx/sites-enabled/THE_FILE)
  sudo cp "$f" "/root/$(basename "$f").bak"
  sudo sed -i 's/ default_server//' "$f"
  ```

Create and enable the catch-all:

```bash
sudo tee /etc/nginx/sites-available/00-catchall >/dev/null <<'EOF'
# Requests for any domain without its own site end here and are dropped, so no
# real project is ever served under a domain it doesn't own.
server {
    listen 80      default_server;
    listen [::]:80 default_server;
    listen 443      ssl default_server;
    listen [::]:443 ssl default_server;
    server_name _;

    ssl_reject_handshake on;   # refuse TLS for unknown names — no cert needed (nginx ≥ 1.19.4)
    return 444;                # close the connection without a response
}
EOF
sudo ln -s /etc/nginx/sites-available/00-catchall /etc/nginx/sites-enabled/00-catchall
sudo nginx -t && sudo systemctl reload nginx
```

Verify:

```bash
curl -sI -H "Host: not-mine.example" http://127.0.0.1    # "Empty reply from server" = working
curl -sI https://bangkok-days.com | head -1              # same as in 1.1
```

## 1.8 Apps directory and port registry

Every project lives in `/srv/apps/<name>`. Its ports are reserved in one file,
so two projects never collide — the most common failure on a shared VM:

```bash
sudo mkdir -p /srv/apps
sudo chown "$USER": /srv/apps
cat > /srv/apps/PORTS.md <<'EOF'
# Port registry — every port is bound to 127.0.0.1; only nginx is public.
# Reserve a block of 100 per project before deploying it.

| Project      | Domains                                       | Frontend | API  | Postgres | Redis | Repo                               |
|--------------|-----------------------------------------------|----------|------|----------|-------|------------------------------------|
| bangkok-days | bangkok-days.com                              | ?        | ?    |          |       |                                    |
| papyrus      | papyrus.anidoes.dev, api.papyrus.anidoes.dev  | 3100     | 8100 | 5732     | 6380  | github.com/anidoesdev/Papyrus      |
| (next)       |                                               | 3200     | 8200 | 5733     | 6381  |                                    |
EOF
```

Fill in bangkok-days' ports from 1.1's `ss` output.

Part 1 is done. The VM now has shared nginx settings, a catch-all, and a port
plan; nothing is project-specific yet.

---

# Part 2 — Deploy Papyrus

## 2.1 Clean up any earlier attempt

If you tried an earlier version of this guide, leftover containers keep the
names this stack needs:

```bash
docker ps -a --format '{{.Names}}' | grep -E '^(rag_|certbot_bootstrap)' || echo "nothing to clean"
```

If anything is listed:

```bash
[ -d ~/scientific-rag-assistant ] && (cd ~/scientific-rag-assistant && docker compose --profile docker-nginx down)
docker rm -f certbot_bootstrap rag_pg rag_redis rag_api rag_frontend rag_nginx rag_certbot 2>/dev/null
sudo rm -f /etc/nginx/sites-enabled/papyrus && sudo nginx -t && sudo systemctl reload nginx
```

Data volumes from an earlier attempt are no longer used. Once you've confirmed
there's nothing in them you need, remove them:

```bash
docker volume ls -q | grep '^scientific-rag-assistant_' | xargs -r docker volume rm
```

## 2.2 Clone

```bash
cd /srv/apps
git clone https://github.com/anidoesdev/Papyrus.git papyrus
cd papyrus
ls data/raw/*.pdf | wc -l        # expect 20 — the papers ship with the repo
```

All remaining Part 2 commands run from `/srv/apps/papyrus`.

## 2.3 Create `.env`

`.env` is git-ignored, so it's created here. The prompts keep the API key out
of your shell history; the database password and JWT secret are generated.

```bash
read -rsp "OpenAI API key: " OPENAI_KEY; echo
read -rp  "Google OAuth client ID (Enter to skip): " GOOGLE_ID

cat > .env <<EOF
# ── Compose / host wiring ─────────────────────────────────────────────
COMPOSE_PROJECT_NAME=papyrus
API_PORT=8100
FRONTEND_PORT=3100

# ── Secrets ───────────────────────────────────────────────────────────
POSTGRES_PASSWORD=$(openssl rand -hex 24)
JWT_SECRET_KEY=$(openssl rand -hex 32)
OPENAI_API_KEY=${OPENAI_KEY}

# ── Google sign-in ────────────────────────────────────────────────────
GOOGLE_CLIENT_ID=${GOOGLE_ID}
NEXT_PUBLIC_GOOGLE_CLIENT_ID=${GOOGLE_ID}

# ── Public URLs (baked into the frontend at build time) ───────────────
NEXT_PUBLIC_API_URL=https://api.papyrus.anidoes.dev
ALLOWED_ORIGINS=https://papyrus.anidoes.dev
EOF

chmod 600 .env
unset OPENAI_KEY GOOGLE_ID
grep -q '^OPENAI_API_KEY=sk-' .env && echo "OpenAI key set" || echo "!! OPENAI_API_KEY is empty — edit .env"
```

What each setting does:

- `COMPOSE_PROJECT_NAME=papyrus` prefixes this stack's network and volumes
  (`papyrus_rag_pg_data`, …), so they never mix with another project's.
- `API_PORT` / `FRONTEND_PORT` are the localhost ports from `PORTS.md`. The
  nginx site in 2.6 must use the same numbers.
- `POSTGRES_PASSWORD` only takes effect when the database volume is first
  created. Changing it later locks the API out of the database.
- `NEXT_PUBLIC_API_URL` is compiled into the JavaScript bundle. Change it and
  you must rebuild the frontend (`docker compose build frontend`); a restart
  isn't enough.
- `ALLOWED_ORIGINS` is FastAPI's CORS allow-list. It must exactly match the
  browser's origin, scheme included. If it doesn't, the site loads but every
  API call fails in the browser while `curl` still works.

If you use Google sign-in, add `https://papyrus.anidoes.dev` under **Authorized
JavaScript origins** in Google Cloud Console → APIs & Services → Credentials.

Because `/api/ask` is public, also set a monthly **spending limit** for the
OpenAI project whose key you used (platform.openai.com → Settings → Limits).

## 2.4 Check the ports are free

```bash
sudo ss -tlnp | grep -E ':(3100|8100|5732|6380)\b' || echo "all free"
```

If anything is using them, pick free ports, put them in `.env` **and**
`PORTS.md`, and use the same numbers in the nginx site in 2.6.
(Postgres 5732 and Redis 6380 are fixed in `docker-compose.yml`.)

## 2.5 Build and start the containers

```bash
docker compose build             # the frontend build takes a few minutes
docker compose up -d             # db + redis start first; api waits for their healthchecks
docker compose ps                # all four should be Up; db and redis (healthy)
```

This starts `db`, `redis`, `api` and `frontend`. The containerised
`nginx`/`certbot` services belong to the `docker-nginx` profile and stay off
(they're for Appendix A only).

All four containers restart automatically after a crash or reboot
(`restart: unless-stopped`). `init.sql` runs on the database's first boot only,
and creates the `vector` extension and the `users` and `chunks` tables.

Check the containers answer locally before involving nginx:

```bash
curl -s http://127.0.0.1:8100/health | python3 -m json.tool      # db / redis / openai: "ok"
curl -s -o /dev/null -w 'frontend: %{http_code}\n' http://127.0.0.1:3100
```

## 2.6 Add the nginx site

The site file ships with the repo (`nginx/host/papyrus.conf`) and uses the
common files from 1.6:

```bash
sudo cp nginx/host/papyrus.conf /etc/nginx/sites-available/papyrus
sudo ln -sf /etc/nginx/sites-available/papyrus /etc/nginx/sites-enabled/papyrus
sudo nginx -t && sudo systemctl reload nginx
```

Test through nginx without waiting for DNS:

```bash
curl -s -H "Host: api.papyrus.anidoes.dev" http://127.0.0.1/health
curl -s -o /dev/null -w 'frontend via nginx: %{http_code}\n' -H "Host: papyrus.anidoes.dev" http://127.0.0.1/
curl -sI https://bangkok-days.com | head -1      # other sites unchanged
```

## 2.7 HTTPS certificate

DNS from 0.1 must be live before this step, or Let's Encrypt validates against
the wrong server:

```bash
dig +short papyrus.anidoes.dev @8.8.8.8          # must print 162.43.26.54
dig +short api.papyrus.anidoes.dev @8.8.8.8      # must print 162.43.26.54
dig +short AAAA papyrus.anidoes.dev @8.8.8.8     # must print nothing
```

Then (use your own email — Let's Encrypt sends expiry warnings there):

```bash
sudo certbot --nginx \
  -d papyrus.anidoes.dev -d api.papyrus.anidoes.dev \
  --redirect -m you@example.com --agree-tos --no-eff-email
```

`certbot --nginx` proves you control the domains through the running nginx,
then edits `/etc/nginx/sites-available/papyrus` in place. It adds the 443
listeners, the certificate paths, and the HTTP→HTTPS redirect. It only touches
server blocks whose `server_name` matches these two domains.

Verify:

```bash
sudo certbot certificates                        # papyrus.anidoes.dev listed, both domains
sudo nginx -t
curl -sI https://papyrus.anidoes.dev | head -1              # HTTP/1.1 200
curl -s  https://api.papyrus.anidoes.dev/health
curl -sI http://papyrus.anidoes.dev | grep -iE '^(HTTP|location)'   # 301 → https
```

`.dev` domains are HSTS-preloaded: browsers refuse plain `http://` for them, so
the site only opens in a browser after this step. `curl` worked before it.

## 2.8 Load the papers

`/app/data` inside the api container is a Docker volume, so the PDFs from the
clone must be copied in:

```bash
docker compose cp data/raw/. api:/app/data/raw/
docker compose exec -T api sh -c 'ls /app/data/raw/*.pdf | wc -l'      # expect 20

docker compose exec -T api python scripts/ingest_all.py --dry-run
docker compose exec -T api python scripts/ingest_all.py                # embeds via OpenAI, a few minutes
```

If the SSH session drops mid-way, just run `ingest_all.py` again — it skips
files already indexed.

## 2.9 Rebuild the vector index — do not skip

`init.sql` builds the vector index on an **empty** table, which gives
near-zero retrieval recall. It has to be rebuilt after the data is loaded,
with `lists ≈ √(number of chunks)`. This computes that and rebuilds in one go:

```bash
docker compose exec -T db psql -U raguser -d ragdb <<'SQL'
DO $$
DECLARE
  n bigint;
  l int;
BEGIN
  SELECT count(*) INTO n FROM chunks;
  IF n = 0 THEN
    RAISE EXCEPTION 'chunks table is empty: run step 2.8 (ingest) first';
  END IF;
  l := round(sqrt(n))::int;
  RAISE NOTICE 'chunks = %, building ivfflat index with lists = %', n, l;
  DROP INDEX IF EXISTS idx_chunks_embedding;
  EXECUTE format(
    'CREATE INDEX idx_chunks_embedding ON chunks
       USING ivfflat (embedding vector_cosine_ops) WITH (lists = %s)', l);
END $$;
ANALYZE chunks;
SQL
```

Expect a notice like `chunks = 350, … lists = 19`. Then clear any answers
cached before the index was fixed:

```bash
docker compose exec -T redis redis-cli FLUSHALL
```

(This is Papyrus's own Redis container, so no other project is affected.)

Repeat 2.9 after **every** re-ingest or `TRUNCATE`.

## 2.10 Final checks

```bash
curl -s https://api.papyrus.anidoes.dev/health | python3 -m json.tool
curl -s -o /dev/null -w 'frontend: %{http_code}\n' https://papyrus.anidoes.dev
curl -sI https://bangkok-days.com | head -1                  # other sites unchanged
```

Rate limit on `/api/ask` (the same question is cached after the first call, so
this costs one OpenAI request):

```bash
for i in $(seq 1 10); do
  curl -s -o /dev/null -w '%{http_code} ' -X POST https://api.papyrus.anidoes.dev/api/ask \
    -H 'Content-Type: application/json' -d '{"question":"What is fine-tuning?"}'
done; echo
# expect several 200s, then 429s
```

Then open **https://papyrus.anidoes.dev** and ask a question about one of the
papers.

Optional reboot test: after `sudo reboot` and reconnecting, `docker ps` should
list all four papyrus containers again, and the site should load, with no
manual steps.

Papyrus is live. 🎉

---

# Part 3 — Adding the next project

Part 1 is already done, so a new project takes seven steps:

```bash
# 1. Reserve a port block — edit /srv/apps/PORTS.md (e.g. 3200 / 8200 / 5733 / 6381)

# 2. DNS: A record  myapp.example.com → 162.43.26.54   (do it first; it takes a while)

# 3. Code + config
cd /srv/apps && git clone <repo-url> myapp && cd myapp
#    .env: COMPOSE_PROJECT_NAME=myapp, plus its own ports and secrets.
#    In its compose file, publish ports as "127.0.0.1:3200:3000" — never bare "3200:3000" —
#    and use container_name values no other project uses (names are VM-wide).

# 4. Start it and check it locally
docker compose up -d --build
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3200

# 5. nginx site from the template
sudo sed -e 's/APP_NAME/myapp/g' -e 's/APP_DOMAIN/myapp.example.com/g' -e 's/APP_PORT/3200/g' \
  /etc/nginx/sites-available/_template.example | sudo tee /etc/nginx/sites-available/myapp >/dev/null
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/myapp
sudo nginx -t && sudo systemctl reload nginx

# 6. HTTPS
sudo certbot --nginx -d myapp.example.com --redirect

# 7. Verify it — and that the other sites are unchanged
curl -sI https://myapp.example.com | head -1
curl -sI https://papyrus.anidoes.dev | head -1
curl -sI https://bangkok-days.com | head -1
```

Projects that aren't containerised (a Node or Python process under systemd)
work the same way, as long as they listen on their reserved `127.0.0.1` port.

**Watch memory.** nginx can route dozens of sites; RAM runs out first. Papyrus
alone uses about 1–1.5 GB (Postgres, Redis, API, Next.js). Check with
`free -h` and `docker stats --no-stream`. To fit more projects, share one
Postgres and one Redis between them (a separate database and user per project)
rather than running a pair per project.

---

# Operations

**Overview of the whole VM**

```bash
docker ps --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'
free -h; df -h /; docker system df
cat /srv/apps/PORTS.md
```

**Logs** (run from `/srv/apps/papyrus`)

```bash
docker compose logs -f api
docker compose logs --tail=100 frontend
sudo tail -f /var/log/nginx/papyrus-api.error.log
sudo tail -f /var/log/nginx/papyrus-api.access.log
```

Container logs are capped at 3 × 10 MB each (`x-logging` in
`docker-compose.yml`), and Ubuntu rotates the nginx logs.

**Deploy a code update**

```bash
cd /srv/apps/papyrus
git pull
docker compose build
docker compose up -d             # recreates only the containers whose image changed
```

**Change the nginx site after certbot.** From 2.7 on, the installed
`/etc/nginx/sites-available/papyrus` contains certbot's HTTPS additions and no
longer matches `nginx/host/papyrus.conf`. To apply a change from the repo:

```bash
sudo cp nginx/host/papyrus.conf /etc/nginx/sites-available/papyrus
sudo certbot --nginx -d papyrus.anidoes.dev -d api.papyrus.anidoes.dev --reinstall --redirect
sudo nginx -t && sudo systemctl reload nginx
```

`--reinstall` re-adds the HTTPS blocks using the existing certificate; it
doesn't request a new one. For a one-line tweak, editing the installed file
directly with `sudo nano` works too.

**Certificates** renew automatically. The certbot package's timer renews every
certificate on the VM and reloads nginx. Check it with:

```bash
systemctl list-timers | grep -i certbot
sudo certbot renew --dry-run
```

**Back up the database**

```bash
cd /srv/apps/papyrus
docker compose exec -T db pg_dump -U raguser ragdb | gzip > ~/papyrus-$(date +%F).sql.gz
```

`-T` matters here. Without it, Docker attaches a terminal, which silently
corrupts the dump's line endings.

**Restore**

```bash
gunzip -c ~/papyrus-YYYY-MM-DD.sql.gz | docker compose exec -T db psql -U raguser -d ragdb
```

**Reset the corpus** (then redo 2.8 and 2.9):

```bash
docker compose exec -T db psql -U raguser -d ragdb -c "TRUNCATE chunks RESTART IDENTITY;"
```

**Free disk space safely**

```bash
docker image prune -f            # removes only untagged leftovers from old builds
```

Avoid `docker system prune -a` on a shared VM. It deletes every image not in
use by a running container, including other projects' images if they happen to
be stopped.

**Take Papyrus offline / remove it** without touching other projects:

```bash
sudo rm /etc/nginx/sites-enabled/papyrus
sudo nginx -t && sudo systemctl reload nginx
cd /srv/apps/papyrus && docker compose down      # add -v to also delete its data volumes
```

---

# Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| `certbot` fails / no `/etc/letsencrypt/live/papyrus…` | DNS still points elsewhere | `dig +short papyrus.anidoes.dev @8.8.8.8` must be `162.43.26.54`; wait out the TTL, retry |
| `certbot`: "Timeout during connect" | Port 80 blocked | `sudo ufw status`; also the provider's control-panel firewall |
| `nginx -t`: "unknown variable ws_connection" | Common files missing | Redo 1.6 |
| `nginx -t`: "duplicate default server" | Two `default_server`s on a port | Redo the check in 1.7 |
| `nginx -t`: "duplicate listen options for [::]:443" | `ipv6only=on` on more than one block | Remove `ipv6only=on` from all but one `listen [::]:443` line |
| **502 Bad Gateway** | Container down or port mismatch | `docker compose ps`; `curl 127.0.0.1:8100/health`; the ports in `.env` and the nginx site must agree |
| **504 Gateway Timeout** on a question | Answer took > 180 s | Raise `proxy_read_timeout` in the api server block |
| **413** on PDF upload | Upload over 50 MB | `client_max_body_size` in the api server block |
| **429** / browser shows a CORS error when asking fast | Hit the `/api/ask` rate limit | Expected. nginx's 429 carries no CORS header, so browsers report it as CORS. Raise `rate`/`burst` in the site's `limit_req_zone` if real users hit it |
| Browser API calls fail, `curl` works | `ALLOWED_ORIGINS` mismatch | Must be exactly `https://papyrus.anidoes.dev`; then `docker compose up -d api` |
| Frontend calls `localhost:8000` | Old bundle | Fix `NEXT_PUBLIC_API_URL` in `.env`, `docker compose build frontend && docker compose up -d` |
| api logs: `password authentication failed` | DB volume was created with a different password | Put the original `POSTGRES_PASSWORD` back in `.env`, or `docker compose down -v` and redo from 2.5 (wipes data) |
| Build dies with exit code 137 | Out of memory | Add swap (1.3), then rebuild |
| "container name … is already in use" | Leftovers from an earlier attempt | 2.1 |
| Questions return "no answer" instantly | Cached miss + bad index | `redis-cli FLUSHALL`, redo 2.9 |
| Wrong site shown for a domain | That domain has no site of its own and there's no catch-all | 1.7 |
| `address already in use` on 80/443 | Started the `docker-nginx` profile | `docker compose --profile docker-nginx stop nginx certbot` |

**Throughput note:** the API runs `uvicorn --workers 1` (see `Dockerfile`).
That limit dates from the old Ollama embedding client. Now that embeddings
come from OpenAI over HTTP, more workers are worth testing if many people use
the site at once — mind the memory.

---

# Appendix A — Dedicated VM with containerised nginx

Only for a fresh VM where **nothing else** uses ports 80/443. nginx and certbot
run as containers (the `docker-nginx` profile) and the host needs no nginx.
Replace `yourdomain.com` with your domain throughout.

```bash
# A1. Certificate — app.conf needs certs to exist, so answer the ACME challenge
#     with a temporary HTTP-only nginx first
mkdir -p certbot/www
docker run --rm -d --name certbot_bootstrap -p 80:80 \
  -v "$PWD/nginx/bootstrap:/etc/nginx/conf.d:ro" \
  -v "$PWD/certbot/www:/var/www/certbot" nginx:alpine
docker run --rm \
  -v "$PWD/certbot/www:/var/www/certbot" -v /etc/letsencrypt:/etc/letsencrypt \
  certbot/certbot certonly --webroot -w /var/www/certbot \
    -d yourdomain.com -d api.yourdomain.com \
    --email you@example.com --agree-tos --no-eff-email
docker stop certbot_bootstrap

# A2. Point the container nginx config at your domain
sed -i 's/yourdomain\.com/REALDOMAIN.com/g' nginx/conf.d/app.conf

# A3. Start everything, including nginx + certbot
docker compose --profile docker-nginx up -d
```

One certificate covers both names, stored under the first `-d` domain — that's
why both server blocks in `app.conf` point at the same path. The `certbot`
container renews every 12 h and the `nginx` container reloads every 12 h. Then
continue from 2.8.

---

# Appendix B — Bare IP (no domain)

Let's Encrypt doesn't issue certificates for IP addresses, so this is
HTTP-only — fine for a demo, not for real sign-ins.

Serve both apps from one origin (no second hostname, no CORS issues). In
`.env`, then rebuild the frontend:

```
NEXT_PUBLIC_API_URL=http://YOUR_VM_IP
ALLOWED_ORIGINS=http://YOUR_VM_IP
```

nginx site:

```nginx
server {
    listen 80;
    server_name YOUR_VM_IP;
    client_max_body_size 50M;

    location /api/   { proxy_pass http://127.0.0.1:8100; include snippets/common-proxy.conf; proxy_read_timeout 180s; }
    location /health { proxy_pass http://127.0.0.1:8100; include snippets/common-proxy.conf; }
    location /       { proxy_pass http://127.0.0.1:3100; include snippets/common-proxy.conf; }
}
```

This works because every backend route the frontend calls is under `/api/`, or
is `/health`.
