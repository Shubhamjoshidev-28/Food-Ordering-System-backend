# Smart Ordering System — Deployment Guide

Reusable runbook for deploying the SOS / Food Ordering System project.
- **Backend + DB:** Oracle Cloud VPS (Ubuntu) — Django + DRF + Gunicorn + Nginx + SQLite
- **Frontend:** GitHub Pages (static HTML/CSS/JS)
- **Domain:** DuckDNS (free) — `sosfooddemo.duckdns.org`
- **SSL:** Let's Encrypt via Certbot

Use this whenever you need to redeploy from scratch, spin up a new VPS, or recover from a broken setup.

---

## 0. Prerequisites

- Oracle Cloud VPS (or any VPS) with Ubuntu, SSH access
- A DuckDNS domain pointed at the VPS's public IP (or your own domain)
- GitHub repo with frontend code, GitHub Pages enabled
- Backend repo cloned locally for reference

---

## 1. VPS Initial Setup

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install python3-venv python3-pip nginx git -y
```

Clone the backend repo:

```bash
git clone <your-backend-repo-url> /var/www/sos
cd /var/www/sos/SOS
```

Create and activate a virtual environment:

```bash
python3 -m venv venv
source venv/bin/activate
pip install -r requirements.txt
pip install gunicorn
```

If `requirements.txt` doesn't exist yet:

```bash
pip install django djangorestframework django-cors-headers gunicorn
pip freeze > requirements.txt
```

Run migrations:

```bash
python manage.py migrate
python manage.py collectstatic --noinput
```

---

## 2. Django `settings.py` — Production Values

```python
DEBUG = False
ALLOWED_HOSTS = ["sosfooddemo.duckdns.org"]

CORS_ALLOW_ALL_ORIGINS = False
CORS_ALLOWED_ORIGINS = [
    "https://shubhamjoshidev-28.github.io",   # origin only, no path/trailing slash
]

MIDDLEWARE = [
    "corsheaders.middleware.CorsMiddleware",   # must sit above CommonMiddleware
    "django.middleware.common.CommonMiddleware",
    # ...rest of your middleware
]
```

**Do not** put the full GitHub Pages path (e.g. `/Food-Ordering-System-Frontend-/`) in `CORS_ALLOWED_ORIGINS` — only scheme + host is checked.

---

## 3. Gunicorn as a systemd Service

Create `/etc/systemd/system/food-backend.service`:

```ini
[Unit]
Description=SOS Django backend (Gunicorn)
After=network.target

[Service]
User=www-data
WorkingDirectory=/var/www/sos/SOS
ExecStart=/var/www/sos/SOS/venv/bin/gunicorn --workers 3 --bind unix:/var/www/sos/SOS/sos.sock SOS.wsgi:application
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
```

Enable and start:

```bash
sudo systemctl daemon-reload
sudo systemctl start food-backend
sudo systemctl enable food-backend
sudo systemctl status food-backend
```

Confirm it's set to survive reboots:

```bash
sudo systemctl is-enabled food-backend   # should print "enabled"
```

---

## 4. Nginx Reverse Proxy

Create `/etc/nginx/sites-available/sos`:

```nginx
server {
    listen 80;
    server_name sosfooddemo.duckdns.org;

    location /static/ {
        alias /var/www/sos/SOS/staticfiles/;
    }

    location / {
        include proxy_params;
        proxy_pass http://unix:/var/www/sos/SOS/sos.sock;
    }
}
```

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/sos /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx
```

---

## 5. Domain (DuckDNS)

1. Create/update the domain at [duckdns.org](https://www.duckdns.org) pointing to your VPS's public IP.
2. Verify DNS resolves:
   ```bash
   nslookup sosfooddemo.duckdns.org
   ```

If your VPS IP changes (common on free-tier VPS restarts), update the DuckDNS IP immediately or the domain will point nowhere.

---

## 6. SSL — Certbot

```bash
sudo apt install certbot python3-certbot-nginx -y
sudo certbot --nginx -d sosfooddemo.duckdns.org
```

Say **yes** to the HTTP→HTTPS redirect prompt.

Verify the cert exists and check expiry:

```bash
sudo certbot certificates
```

Confirm auto-renewal is set up (Certbot installs this automatically):

```bash
sudo systemctl status certbot.timer
sudo certbot renew --dry-run
```

---

## 7. Firewall — Two Separate Layers (both must be open)

Oracle Cloud (and some other providers) block traffic at **two independent layers**. Both need port 443 (and 80, 22) open, or you'll get connection failures that look like backend bugs but aren't.

### 7a. OCI Security List (cloud-level firewall)

**OCI Console → Networking → Virtual Cloud Networks → your VCN → Subnets → Security List → Ingress Rules**

Add/confirm these rules:

| Source | Protocol | Destination Port |
|---|---|---|
| 0.0.0.0/0 | TCP | 22 |
| 0.0.0.0/0 | TCP | 80 |
| 0.0.0.0/0 | TCP | 443 |

### 7b. iptables (VM-level firewall)

Even if `ufw status` shows `inactive`, Oracle's default Ubuntu images often ship with a separate active `iptables` ruleset that `ufw` doesn't manage.

Check current rules:

```bash
sudo iptables -L INPUT -n --line-numbers
```

If there's a `REJECT` rule at the bottom and no explicit `ACCEPT` for 443/80 above it, insert one (adjust the line number to sit *before* the REJECT rule):

```bash
sudo iptables -I INPUT <line-before-reject> -p tcp --dport 443 -j ACCEPT
sudo iptables -I INPUT <line-before-reject> -p tcp --dport 80 -j ACCEPT
```

Make it persist across reboots:

```bash
sudo apt install iptables-persistent -y
sudo netfilter-persistent save
```

**Symptom cheat sheet:**
- `curl` hangs / times out → likely OCI Security List blocking (silent drop)
- `curl` returns `No route to host` → likely local `iptables` REJECT rule
- `curl` returns `Connection refused` → nothing listening on that port (check Nginx/Gunicorn)

---

## 8. Verify Backend End-to-End

```bash
curl -Iv https://sosfooddemo.duckdns.org/demo/menu_list/
```

Expect: successful TLS handshake, valid cert, `HTTP/1.1 200 OK`.

---

## 9. Frontend — GitHub Pages

In `js/api.js`, confirm the base URL includes the `/demo` prefix:

```javascript
const BASE_URL = "https://sosfooddemo.duckdns.org/demo";
```

Grep to make sure no other file hardcodes a stale URL:

```bash
grep -rn "duckdns" src/js/
```

Push to the branch GitHub Pages serves from (`gh-pages` or `main`/`docs`, depending on your repo settings):

```bash
git add .
git commit -m "Update API base URL"
git push origin main
```

Live site: `https://shubhamjoshidev-28.github.io/Food-Ordering-System-Frontend-/`

GitHub Pages deploys typically take 30–90 seconds. Hard-refresh (Ctrl+Shift+R) to bypass browser JS caching after redeploying.

---

## 10. Post-Deploy Test Checklist

Run these against the **live** URL, not localhost:

- [ ] Menu list loads (`GET /demo/menu_list/`)
- [ ] Create order
- [ ] Update order — status-only
- [ ] Update order — with new items (recomputes `Total`)
- [ ] Update order status to **`Delivered`** specifically (known bug path — see Known Issues)
- [ ] Delete order
- [ ] Add/edit menu item
- [ ] Invoice generation renders correctly
- [ ] Browser console shows no CORS errors on the live GitHub Pages site

---

## 11. Known Issues to Fix in Code (carry over until patched)

- `Order_Service.get_order(status)` raises `UnboundLocalError` when `status == "Delivered"` — `order` variable only assigned in the `if` branch.
- `Staff` field on `Order` model has no `max_length` — will fail validation/migrations until one is added.
- Keep `requirements.txt` up to date (`pip freeze > requirements.txt` after adding any new package).

---

## 12. Quick Redeploy Checklist (after first-time setup is done)

When you just need to push new backend code:

```bash
cd /var/www/sos/SOS
git pull
source venv/bin/activate
pip install -r requirements.txt
python manage.py migrate
sudo systemctl restart food-backend
```

When you just need to push new frontend code:

```bash
git add . && git commit -m "update" && git push origin main
```
(GitHub Pages auto-rebuilds — no server action needed)

---

## 13. Useful Diagnostic Commands

```bash
# Backend service status/logs
sudo systemctl status food-backend
sudo journalctl -u food-backend -f

# Nginx status/logs
sudo systemctl status nginx
sudo tail -f /var/log/nginx/error.log

# Confirm port 443 is listening
sudo ss -tlnp | grep 443

# Confirm cert validity
sudo certbot certificates

# Confirm both firewall layers
sudo ufw status
sudo iptables -L INPUT -n --line-numbers
```