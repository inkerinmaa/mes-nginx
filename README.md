# mes-nginx

Nginx TLS-termination layer for the MES local development stack running on WSL2.

## What this does

```
Browser (Windows)
    |
    | HTTPS (*.test.local, port 443)
    v
nginx (WSL2, this config)        ← TLS terminates here
    |
    +-- mes.test.local   →  localhost:8090  (Docker: mes-frontend nginx)
    |                          mes-frontend nginx proxies /api/ and /hubs/ to mes-backend:5000
    |
    +-- keycloak.test.local  →  localhost:8080  (Docker: Keycloak)
    |
    +-- nifi.test.local      →  localhost:8443  (Docker: NiFi, HTTPS internally)
```

Nginx here does **only TLS termination**. All application routing (SPA, `/api/`, `/hubs/`, SignalR) is handled by the inner nginx inside the Docker container.

## Prerequisites

- WSL2 with Ubuntu (20.04 / 22.04 / 24.04)
- Docker Desktop with WSL2 backend enabled
- The following Docker stacks already set up:
  - `~/projects/keycloak-setup/` — Keycloak + PostgreSQL (port 8080)
  - `~/projects/mes-docker/` — Redis + mes-backend + mes-frontend/nginx (port 8090)
  - (optional) NiFi on port 8443

---

## Step 1 — Install nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable nginx
```

---

## Step 2 — Generate TLS certificates

Certificates live in `~/projects/ssl-certs/`. A wildcard cert for `*.test.local` is used for all services.

```bash
cd ~/projects/ssl-certs
chmod +x generate-certs.sh
./generate-certs.sh
```

This creates:
- `ca.crt` — local Certificate Authority (install once in trust stores)
- `test.local.crt` / `test.local.key` — wildcard cert used by nginx

See `~/projects/ssl-certs/README.md` for full details.

### Trust the CA on WSL (so .NET and curl trust *.test.local)

```bash
sudo cp ~/projects/ssl-certs/ca.crt /usr/local/share/ca-certificates/local-dev-ca.crt
sudo update-ca-certificates
```

### Trust the CA on Windows (so the browser trusts *.test.local)

Open PowerShell **as Administrator**:

```powershell
# Adjust distro name if needed (Ubuntu, Ubuntu-24.04, etc.)
$certPath = "\\wsl$\Ubuntu\home\nik\projects\ssl-certs\ca.crt"
Import-Certificate -FilePath $certPath -CertStoreLocation Cert:\LocalMachine\Root
```

Or manually: double-click `ca.crt` in Windows Explorer → Install Certificate → Local Machine → Trusted Root Certification Authorities.

Restart the browser after trusting.

---

## Step 3 — Install nginx site configs

```bash
# Copy site configs
sudo cp ~/projects/mes-nginx/sites-available/mes.test.local      /etc/nginx/sites-available/
sudo cp ~/projects/mes-nginx/sites-available/keycloak.test.local /etc/nginx/sites-available/
sudo cp ~/projects/mes-nginx/sites-available/nifi.test.local     /etc/nginx/sites-available/

# Enable sites (remove default if it conflicts on port 80)
sudo ln -sf /etc/nginx/sites-available/mes.test.local      /etc/nginx/sites-enabled/
sudo ln -sf /etc/nginx/sites-available/keycloak.test.local /etc/nginx/sites-enabled/
sudo ln -sf /etc/nginx/sites-available/nifi.test.local     /etc/nginx/sites-enabled/

# Test config, then reload
sudo nginx -t && sudo systemctl reload nginx
```

> If you don't run NiFi, skip `nifi.test.local`.

### Cert path

The configs reference `/home/nik/projects/ssl-certs/`. If deploying to a different user, update the `ssl_certificate` and `ssl_certificate_key` lines in each site file, or symlink:

```bash
# Alternative: copy certs to /etc/ssl
sudo cp ~/projects/ssl-certs/test.local.crt /etc/ssl/certs/test.local.crt
sudo cp ~/projects/ssl-certs/test.local.key /etc/ssl/private/test.local.key
sudo chmod 640 /etc/ssl/private/test.local.key
sudo chown root:www-data /etc/ssl/private/test.local.key
# Then update ssl_certificate paths in site configs accordingly.
```

---

## Step 4 — DNS (hosts file)

### Windows hosts file

Open Notepad **as Administrator**, edit `C:\Windows\System32\drivers\etc\hosts`:

```
# MES local development
127.0.0.1   test.local
127.0.0.1   mes.test.local
127.0.0.1   api.test.local
127.0.0.1   keycloak.test.local
127.0.0.1   nifi.test.local
```

Flush DNS cache:

```powershell
ipconfig /flushdns
```

### WSL hosts file

```bash
echo '127.0.0.1 mes.test.local api.test.local keycloak.test.local nifi.test.local' | sudo tee -a /etc/hosts
```

Prevent WSL from overwriting `/etc/hosts` on restart — add to `/etc/wsl.conf`:

```ini
[network]
generateHosts = false
```

---

## Step 5 — Start the Docker stacks

```bash
# Keycloak + PostgreSQL
cd ~/projects/keycloak-setup && docker compose up -d

# MES stack (Redis + backend + frontend)
cd ~/projects/mes-docker && docker compose up -d --build
```

Wait ~15 seconds for Keycloak to be ready before testing.

---

## Step 6 — Verify

```bash
# nginx config is valid
sudo nginx -t

# nginx is running
sudo systemctl status nginx

# TLS handshake works (requires CA trusted in WSL)
curl -v https://mes.test.local 2>&1 | grep -E "SSL|HTTP|< "
curl -v https://keycloak.test.local/realms/mes-realm/.well-known/openid-configuration 2>&1 | grep -E "SSL|issuer"

# Docker containers are up
cd ~/projects/mes-docker && docker compose ps
cd ~/projects/keycloak-setup && docker compose ps
```

---

## nginx management

```bash
sudo systemctl start nginx      # start
sudo systemctl stop nginx       # stop
sudo systemctl reload nginx     # reload config without dropping connections
sudo systemctl restart nginx    # full restart

sudo nginx -t                   # test config syntax before reload

# Logs
sudo tail -f /var/log/nginx/mes.test.local.access.log
sudo tail -f /var/log/nginx/mes.test.local.error.log
sudo tail -f /var/log/nginx/keycloak.test.local.error.log
sudo tail -f /var/log/nginx/error.log   # global nginx errors
```

---

## Port map

| Service | nginx listens | Forwards to | Served by |
|---------|---------------|-------------|-----------|
| `mes.test.local` | 443 | `localhost:8090` | Docker: mes-frontend nginx (SPA + API proxy) |
| `api.test.local` | 443 | `localhost:8090` | same Docker container |
| `keycloak.test.local` | 443 | `localhost:8080` | Docker: Keycloak |
| `nifi.test.local` | 443 | `localhost:8443` | Docker: NiFi (HTTPS) |

The port `8090` for the MES frontend is set by `HTTP_PORT` in `~/projects/mes-docker/.env`. Change that value and update the `proxy_pass` in `mes.test.local` if you use a different port.

---

## Adding a new service

1. Create `sites-available/<name>.test.local` following an existing config as template.
2. Add the domain to the Windows and WSL hosts files.
3. `sudo ln -sf /etc/nginx/sites-available/<name>.test.local /etc/nginx/sites-enabled/`
4. `sudo nginx -t && sudo systemctl reload nginx`

No cert work needed — the wildcard `*.test.local` cert already covers new subdomains.

---

## Production / real server

On a server with real DNS and public IPs:

1. Replace self-signed certs with **Let's Encrypt**:
   ```bash
   sudo apt install certbot python3-certbot-nginx
   sudo certbot --nginx -d mes.example.com -d keycloak.example.com
   ```
2. Update `server_name` in each site config to match your real domain.
3. Update `KC_HOSTNAME` in the Keycloak compose and `KEYCLOAK_AUTHORITY` in `mes-docker/.env` to match the new domain.
4. The `proxy_pass` targets (`localhost:8080`, `localhost:8090`) stay the same if Docker runs on the same host.
