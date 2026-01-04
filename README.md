# ZITADEL Docker (V1 first → then enable Login V2)

This repo documents a reliable “get it working first” approach:

1) **Boot ZITADEL with Login V1** (stable)  
2) Sign in to the **V1 Console**  
3) Create a **Service User** with the **IAM Login Client** role  
4) Create a **PAT** and save it as `login-client.pat`  
5) Switch to the **V2 compose** and enable Login V2

It’s written to be **step-by-step**, copy/paste friendly, and safe to share (anonymised placeholders).

---

## What you need before you start

- Docker / Docker Compose on your server
- A working reverse proxy that terminates TLS (example: Nginx Proxy Manager)
- A reachable Postgres (external), already configured for ZITADEL
- A domain name pointing to your reverse proxy

> This guide assumes TLS is handled by the reverse proxy, so ZITADEL runs with `--tlsMode external`.

---

## Repo files you will have

You should have these in your repo (examples provided):

- `docker-compose.v1.yml` (Login V1 initial install)
- `docker-compose.v2.yml` (Login V2 + login container)
- `zitadel-secrets.example.yaml`
- `zitadel-init-steps.example.yaml`
- `.gitignore`

You will create these local copies (do **not** commit real secrets):

- `zitadel-secrets.yaml`
- `zitadel-init-steps.yaml`
- `login-client.pat` (after you generate it)

---

## Step 0 — Copy example YAML files into “real” filenames

In your repo directory:

```bash
cp zitadel-secrets.example.yaml zitadel-secrets.yaml
cp zitadel-init-steps.example.yaml zitadel-init-steps.yaml
```

Now edit the real files:

```bash
nano zitadel-secrets.yaml
nano zitadel-init-steps.yaml
```

### `zitadel-secrets.yaml` (what goes in here)

This file contains your DB credentials (admin + runtime user). Example structure:

```yaml
Database:
  Postgres:
    Admin:
      Username: "PG_ADMIN_USER"
      Password: "PG_ADMIN_PASSWORD"
    User:
      Username: "zitadel"
      Password: "ZITADEL_DB_PASSWORD"
```

### `zitadel-init-steps.yaml` (what goes in here)

This file controls the first admin user created on first init:

```yaml
FirstInstance:
  Org:
    Name: "MyOrg"
    Human:
      Username: "admin"
      Password: "ChangeMeStrong!"
      PasswordChangeRequired: false
```

---

## Step 1 — Export the master key (VERY IMPORTANT)

ZITADEL requires a **32-byte masterkey**. If it’s not 32 bytes, you will get errors.

Create and export a 32-char key (ASCII):

```bash
export ZITADEL_MASTERKEY="0123456789abcdef0123456789abcdef"
```

Sanity check length:

```bash
python3 - << 'PY'
import os
print(len(os.environ.get("ZITADEL_MASTERKEY","")))
PY
```

It must print **32**.

> Keep this key safe. If you lose it, you can’t decrypt stored secrets.

---

## Step 2 — Reverse proxy (must exist before you go public)

Your proxy must forward to ZITADEL and (later) the Login V2 container.
### Details: Scheme=> *http* | forward host => *zitadel IP* | Forward Port *8080*
---
### Custom Locations
--
#### Custom Location 1
  ##### Location: */ui/v2/login*
  ##### *http* | *zitadel ip* | port *3000*
#### Custom Location 2
  ##### Location:*/ui/v2/login/_next*
  ##### *http* | *zitadel ip* | port *3000*
--
Recommended headers:

```nginx
proxy_set_header Host $host;
proxy_set_header X-Forwarded-Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Port $server_port;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
```

### Proxy routes you need

- Main host: forward `https://<AUTH_DOMAIN>` → `http://<DOCKER_HOST>:8080`
- Later (V2): custom location `/ui/v2/login` → `http://<DOCKER_HOST>:3000`

---

# Part A — Boot with Login V1 first (stable)

## Step 3 — Use the V1 compose (initial install)

Create `docker-compose.v1.yml` chnage information to yours.

### `docker-compose.v1.yml`

> This starts ZITADEL using **start-from-init** and keeps **Login V2 disabled**.

```yaml
services:
  zitadel:
    restart: unless-stopped
    image: ghcr.io/zitadel/zitadel:latest
    command: >
      start-from-init
      --masterkey "${ZITADEL_MASTERKEY}"
      --tlsMode external
      --config /current-dir/zitadel-secrets.yaml
      --steps /current-dir/zitadel-init-steps.yaml

    environment:
      # Public URL (users come via HTTPS reverse proxy)
      #<AUTH_DOMAIN>  can be example.domain.com
      ZITADEL_EXTERNALDOMAIN: <AUTH_DOMAIN> 
      ZITADEL_EXTERNALPORT: "443"
      ZITADEL_EXTERNALSECURE: "true"

      # External Postgres
      ZITADEL_DATABASE_POSTGRES_HOST: <PG_HOSTNAME_OR_IP>
      ZITADEL_DATABASE_POSTGRES_PORT: "5432"
      ZITADEL_DATABASE_POSTGRES_DATABASE: zitadel
      ZITADEL_DATABASE_POSTGRES_SSL_MODE: require

      # Disable Login V2 for initial stable setup
      ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_REQUIRED: "false"

    user: "0"
    volumes:
      - .:/current-dir:delegated
    ports:
      - "8080:8080"
    networks:
      - zitadel

networks:
  zitadel:
```

## Step 4 — Start V1

From the repo folder:

```bash
docker compose -f docker-compose.v1.yml up -d
docker ps
```

Tail logs until it settles:

```bash
docker logs -f zitadel
```

---

## Step 5 — Sign in to the Console (V1)

Open in your browser:

- Console: `https://<AUTH_DOMAIN>/ui/console`

Login with the admin created in `zitadel-init-steps.yaml`.

Your login name format is usually:

`<admin_username>@<org_name>.<AUTH_DOMAIN>`

Example:
- `admin@zitadel.<AUTH_DOMAIN>`

---

# Part B — Create the login PAT (so V2 can work)

Login V2 needs a service user token (PAT) that has the **IAM Login Client** role.

## Step 6 — Create a Service User (Machine User)

In the Console:

1. Go to **Users**
2. Click **New**
3. Choose **Service User / Machine User**
4. Create it (name it something obvious like `login-client`)

## Step 7 — Give it the correct role

Still in the Console:

1. Go to **Instance** settings (this done by clicking on "Default Settings at top right)
2. Go to **Members**
3. Add your service user
4. Assign role: **IAM Login Client**
5. Save

## Step 8 — Create a PAT and save it as `login-client.pat`

1. Open the service user
2. Find **Personal Access Tokens**
3. Create a new token
4. Copy the token value
5. On your server, same location as docker compose file, create the file:

```bash
nano login-client.pat
```

Paste the token (one line), save, exit.

Lock down permissions:

```bash
chmod 600 login-client.pat
```

---

# Part C — Switch to Login V2

## Step 9 — Stop V1 and start V2 compose

Stop V1 container (data remains in Postgres):

```bash
docker compose -f docker-compose.v1.yml down
```

Create `docker-compose.v2.yml` with the anonymised config below.

### `docker-compose.v2.yml` (anonymised)

> This uses **start** (not init) and runs the `zitadel-login` container on port 3000.

```yaml
services:
  zitadel:
    restart: unless-stopped
    image: ghcr.io/zitadel/zitadel:latest
    command: >
      start
      --masterkey "${ZITADEL_MASTERKEY}"
      --tlsMode external
      --config /current-dir/zitadel-secrets.yaml

    environment:
      # Public URL (users come via HTTPS reverse proxy)
      ZITADEL_EXTERNALDOMAIN: <AUTH_DOMAIN>
      ZITADEL_EXTERNALPORT: "443"
      ZITADEL_EXTERNALSECURE: "true"

      # External Postgres
      ZITADEL_DATABASE_POSTGRES_HOST: <PG_HOSTNAME_OR_IP>
      ZITADEL_DATABASE_POSTGRES_PORT: "5432"
      ZITADEL_DATABASE_POSTGRES_DATABASE: zitadel
      ZITADEL_DATABASE_POSTGRES_SSL_MODE: require

      # Enable Login V2
      ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_REQUIRED: "true"
      ZITADEL_DEFAULTINSTANCE_FEATURES_LOGINV2_BASEURI: "https://<AUTH_DOMAIN>/ui/v2/login"

      # Public redirect helpers (must be the public domain, not localhost)
      ZITADEL_OIDC_DEFAULTLOGINURLV2: "https://<AUTH_DOMAIN>/ui/v2/login/login?authRequest="
      ZITADEL_OIDC_DEFAULTLOGOUTURLV2: "https://<AUTH_DOMAIN>/ui/v2/login/logout?post_logout_redirect="
      ZITADEL_SAML_DEFAULTLOGINURLV2: "https://<AUTH_DOMAIN>/ui/v2/login/login?samlRequest="

    user: "0"
    volumes:
      - .:/current-dir:delegated
    ports:
      - "8080:8080"
    networks:
      - zitadel

  login:
    restart: unless-stopped
    image: ghcr.io/zitadel/zitadel-login:latest
    environment:
      ZITADEL_API_URL: http://zitadel:8080
      NEXT_PUBLIC_BASE_PATH: /ui/v2/login
      ZITADEL_SERVICE_USER_TOKEN_FILE: /current-dir/login-client.pat
      CUSTOM_REQUEST_HEADERS: Host:<AUTH_DOMAIN>
    user: "0"
    volumes:
      - .:/current-dir:ro
    ports:
      - "3000:3000"
    networks:
      - zitadel

networks:
  zitadel:
```

## Step 10 — Start V2

```bash
docker compose -f docker-compose.v2.yml up -d
docker ps
```

Watch login logs:

```bash
docker logs -f zitadel-login
```

---

## Step 11 — Enable Login V2 in the UI (Default Settings)

In the Console:

1. Go to **Default Settings** (or Instance Settings)
2. Find **Login V2**
3. Enable it
4. Set Base URI: `https://<AUTH_DOMAIN>/ui/v2/login`
5. Save

> Keep your existing admin session open in one browser while testing V2 in another browser.

---

## Step 12 — Test Login V2 in a second browser

Open:

- `https://<AUTH_DOMAIN>/ui/v2/login/`

Sign in with the admin user defined in `zitadel-init-steps.yaml`.

---

# Quick troubleshooting

## PAT not working / login container stuck
- Confirm file exists and is non-empty:

```bash
ls -l login-client.pat
wc -c login-client.pat
```

- Confirm permissions:

```bash
chmod 600 login-client.pat
```

- Restart login:

```bash
docker compose -f docker-compose.v2.yml restart login
docker logs -f zitadel-login
```

## 504 Gateway Timeout (reverse proxy)
Your proxy can’t reach upstream ports.

```bash
curl -I http://127.0.0.1:8080/debug/healthz
curl -I http://127.0.0.1:3000/ui/v2/login/
```

Fix proxy host / custom location.

---

# Safety notes

- Do **not** commit `login-client.pat` or real secrets to GitHub.
- Keep `zitadel-secrets.yaml` private (or use a secret manager).
- Keep your masterkey safe forever.

---

## Copy/paste checklist

V1 bring-up:

```bash
cp zitadel-secrets.example.yaml zitadel-secrets.yaml
cp zitadel-init-steps.example.yaml zitadel-init-steps.yaml
export ZITADEL_MASTERKEY="0123456789abcdef0123456789abcdef"
docker compose -f docker-compose.v1.yml up -d
docker logs -f zitadel
```

Create PAT:

```bash
nano login-client.pat
chmod 600 login-client.pat
```

Switch to V2:

```bash
docker compose -f docker-compose.v1.yml down
docker compose -f docker-compose.v2.yml up -d
docker logs -f zitadel-login
```
# Feel free to contribute back to the repo, there is room for automation after initial learning has been achieved.