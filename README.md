# ByteStorm - Cloud & Infrastructure Department

Server documentation for the project **Smart Greenhouse with Remote Control**.
July - October 2026.

**Status: All stages (1-5) delivered. Server live over HTTPS.**

> **For the other departments:** the server is live at **https://sera.bytestorm.site**
> (and at http://169.58.12.143). API at https://sera.bytestorm.site/api/v1/.
> What you can do now and what comes next - section 2.

---

## 1. Team

| Role | Responsibilities |
|---|---|
| Cloud & Infrastructure | Server, OS, security, MySQL, NGINX, SSL, deployment |
| Coordinator | Access to the Contabo panel, decisions, escalation |

---

## 2. Current status

Last updated: **9 September 2026**

### Stage 1 - Server preparation: DELIVERED

| Deliverable | Acceptance criterion | Status |
|---|---|---|
| Server reachable | IP responds in a browser during Week 1 | OK |
| SSH connection | Successful login, IP shared with the team | OK |
| System updated | No errors during the update | OK - kernel 5.14.0-687.26.1 |
| Non-root user | Working user, root disabled for SSH | OK - `admin` + `sudo` |
| Firewall configured | Only ports 22, 80, 443 open | OK - verified `enabled` at boot |
| NGINX installed | Default page reachable via IP | OK |

### "Canopy" landing page: DELIVERED

Static presentation page (build 1.0.0). Live at https://sera.bytestorm.site.
Deployment procedure in section 7.

### Stage 2 - Database: DELIVERED

| Deliverable | Acceptance criterion | Status |
|---|---|---|
| MySQL Server installed | Service started and active | OK - MySQL 8.0.46 |
| Hardening | Root password set, test DB removed | OK - `mysql_secure_installation` |
| Project database | Visible with `SHOW DATABASES` | OK - `sera_db` |
| Dedicated user | Access only to the project database | OK - `sera_user` |
| phpMyAdmin | Access with the created account (not root) | OK |
| Credentials to Backend | Backend can connect to the DB | OK - delivered |

### Stage 3 - Backend: DELIVERED

| Step | Status |
|---|---|
| 3.1 Node.js LTS + npm | OK - Node 22.23.1, npm 10.9.8 |
| 3.2 PM2 global | OK - PM2 7.0.3, auto-start at boot |
| 3.3 Minimal Express test | OK - working |
| 3.4 Clone backend code | OK - full deployment |
| 3.5 Configure `.env` | OK - production .env on the server |

Continuous deployment: each new version from Backend is pulled with `git pull` +
`pm2 restart`. Backend is live through an NGINX reverse proxy (/api/, /health, /ws).

### Stage 4 - Frontend: DELIVERED (reverse proxy)

NGINX proxies /api/, /health and /ws to the backend. The frontend reaches the API
through https://sera.bytestorm.site/api/v1/. Serving the static React build from
NGINX is enabled once Frontend delivers a build (flow in section 7).

### Stage 5 - HTTPS: DELIVERED

| Deliverable | Acceptance criterion | Status |
|---|---|---|
| Domain | Domain reachable | OK - sera.bytestorm.site -> 169.58.12.143 |
| Certbot installed | Certbot installed successfully | OK - certbot 3.1.0 |
| SSL certificate | App reachable over https:// | OK - valid until 2026-12-08 |
| Auto-renewal | Certificate valid, expiry in the future | OK - certbot-renew.timer active |

---

## 3. Infrastructure

| Item | Value |
|---|---|
| Provider | Contabo |
| Hostname | `vmi3438544.contaboserver.net` |
| Public IP | `169.58.12.143` |
| Domain | **sera.bytestorm.site** (HTTPS active, certificate auto-renewed) |
| Operating system | AlmaLinux 9.8 (Olive Jaguar) |
| Kernel | 5.14.0-687.26.1.el9_8.x86_64 |
| Working user | `admin` (group `wheel`, uid 1001) |
| Access | SSH port 22, **key-only (ed25519)** |
| Active period | July - October 2026 |

### Connecting

```bash
ssh admin@169.58.12.143
```

Password authentication is **disabled**. Direct `root` login is **disabled**.
For access, send your SSH public key (`~/.ssh/id_ed25519.pub`) to the Cloud lead.

### Emergency access

The Contabo panel provides a VNC console and rescue mode if SSH is unavailable.

---

## 4. System architecture

```
                    +---------------------------------------------+
                    |              VPS SERVER                     |
Browser  --HTTPS-->|  +--------+                                 |
user        :443    |  |        |--> /              Canopy  OK    |
                    |  |        |--> /db-sera-2026  phpMyAdmin OK |
Raspberry           |  | NGINX  |        +--> PHP-FPM (socket)    |
Pi Pico 2W          |  |   OK   |--> /api/  --> Node.js  OK       |
                    |  +--------+     proxy      :3000            |
                    |                              |              |
                    |                              v              |
                    |                        MySQL :3306  OK      |
                    |                        (local only)         |
                    +---------------------------------------------+
```

All external traffic enters through NGINX, now over HTTPS (443). Node.js and MySQL
are reachable **only from the server itself**, via `localhost`.

> HTTP (port 80) automatically redirects to HTTPS (443).

### Ports

| Port | Service | Firewall | Reason |
|---|---|---|---|
| 22 | SSH | Open | Administration |
| 80 | HTTP | Open | Redirect to HTTPS + Certbot renewal |
| 443 | HTTPS | Open | Secure traffic (active) |
| 3000 | Node.js | Closed | Reached only through the NGINX proxy |
| 3306 | MySQL | Closed | Local access only |
| 9090 | Cockpit | Removed | Unused web interface |

Check: `sudo firewall-cmd --list-all`

---

## 5. Installed services

| Service | Version | Role | Starts at boot |
|---|---|---|---|
| NGINX | 1.20.1 | Web server, reverse proxy, SSL | Yes - `systemctl enable` |
| MySQL | 8.0.46 | Database | Yes - `systemctl enable` |
| PHP-FPM | 8.2.32 | PHP execution (phpMyAdmin) | Yes - `systemctl enable` |
| PM2 | 7.0.3 | Node.js process manager | Yes - `pm2-admin` service |
| Node.js | 22.23.1 | Backend runtime | Runs under PM2 |
| Certbot | 3.1.0 | SSL certificates + renewal | Yes - `certbot-renew.timer` |
| firewalld | - | Firewall | Yes - `systemctl enable` |

**All verified after a test reboot** - they come back automatically, no manual action.

---

## 6. Database

| Item | Value |
|---|---|
| Engine | MySQL 8.0.46 |
| Database | `sera_db` |
| Encoding | `utf8mb4` / `utf8mb4_unicode_ci` (diacritics supported) |
| Application user | `sera_user`@`localhost` |
| Privileges | `ALL PRIVILEGES` on `sera_db` only |
| External access | No - port 3306 closed in the firewall |

Additional users: `sera_front` (SELECT, Frontend), `sera_hard` (ALL, Hardware),
all `@localhost`, limited to `sera_db`.

### For Backend - `.env` variables

```
DB_HOST=localhost
DB_PORT=3306
DB_NAME=sera_db
DB_USER=sera_user
DB_PASSWORD=<shared over a private channel>
```

> **`.env` is never committed to GitHub.** Commit a `.env.example` with keys and empty
> values instead. The production `.env` is created on the server (it also holds
> JWT_SECRET and the other locally generated secrets, NODE_ENV=production).

### phpMyAdmin

Reachable at https://sera.bytestorm.site/db-sera-2026, with the `sera_user` credentials.

**The path is deliberately not the default (`/phpmyadmin`).** Automated scanners probe
the standard path on any public server; an unusual path filters them out effortlessly.
The address is not posted anywhere public.

---

## 7. Deployment

### Backend Node.js (continuous procedure)

```bash
cd ~/backend
git pull                          # asks for GitHub user + token as the password
# npm ci                          # only if package.json changed
pm2 restart bytestorm-backend
pm2 save                          # after any add/remove of an app
curl -i http://localhost:3000/health   # check -> status ok
```

> **`pm2 save` is not optional.** Without it, the app will not come back in the
> correct state on the next server reboot.

### Static page (procedure tested on the Canopy landing page)

```bash
# 1. On the local machine: transfer to a temporary staging area
scp -r LOCAL_PATH\* admin@169.58.12.143:/tmp/upload/

# 2. On the server: move + permissions + SELinux
sudo mv /tmp/upload/* /var/www/SITE/
sudo chown -R nginx:nginx /var/www/SITE
sudo chmod -R 755 /var/www/SITE
sudo restorecon -Rv /var/www/SITE

# 3. NGINX configuration (one time only)
sudo nginx -t && sudo systemctl reload nginx
```

**Notes:**
- The transfer goes through `/tmp` because `admin` cannot write directly to `/var/www`.
- `restorecon` is mandatory - without it SELinux blocks access (403 error).
- To verify, use `curl` on the server or an incognito window (to avoid the cache).

Current site: **Canopy landing** in `/var/www/canopy`.

### React app (Stage 4)

When Frontend delivers a build: `git pull` + `npm run build`, then NGINX serves the
`dist/` folder. The `/api/` reverse proxy is already configured.

### Superadmin (via the official CLI, NOT direct SQL)

```bash
cd ~/backend
npm run admin:create -- --nume <NAME> --prenume <FIRSTNAME> --email <EMAIL> --parola <PASSWORD>
```

**Important:** changing the role directly in SQL (`UPDATE utilizator SET rol=...`) is
NOT enough - the app keeps seeing the user as `utilizator`. The role must be granted
through the official CLI, which runs the full application flow. The CLI only works if
there is no active superadmin already.

---

## 8. Monitoring

| Check | Command | Frequency |
|---|---|---|
| Active services | `systemctl status nginx mysqld php-fpm --no-pager` | Weekly |
| Node apps | `pm2 list` | Weekly |
| NGINX error logs | `sudo tail -50 /var/log/nginx/error.log` | On issues |
| NGINX access logs | `sudo tail -f /var/log/nginx/access.log` | Client debugging |
| App logs | `pm2 logs bytestorm-backend` | On issues |
| Disk space | `df -h` | Weekly |
| Firewall | `sudo firewall-cmd --list-all` | After changes |
| SSL certificate | `sudo certbot certificates` | Monthly |
| System updates | `sudo dnf check-update` | Monthly |

> Sensor readings accumulate in the database - keep an eye on disk space.
>
> **Troubleshooting "the API returns 404 / is down":** first check that the process is
> running (`pm2 list` + `curl localhost:3000/health`). If the server answers correctly,
> check the access log to see whether the client request actually reaches the server -
> often the problem is on the client side (wrong base URL, proxy, multiple instances),
> not on the server.

---

## 9. Security rules

1. **Never work as `root`.** Use `admin` + `sudo`.
2. **No passwords in this repo.** Not in code, not in issues. Git history never forgets.
3. **`.env` files are never committed.** See `.gitignore`.
4. **The MySQL port (3306) is never opened in the firewall.** Ever.
5. **Credentials are shared over a private channel**, directly with the person who needs them.
6. **The phpMyAdmin address is never posted publicly.**
7. **HTTPS is active** - all traffic is encrypted, HTTP redirects to HTTPS.

### If a password accidentally ends up in the repo

Deleting it in a new commit is not enough. Notify the Cloud department immediately -
the password must be **changed on the server**; the old version stays in history.

### Why password authentication is fully disabled

On the first connection, the banner reported **8450 failed automated login attempts**
since the last successful login. The brief asked for "root disabled for SSH"; we went
one step further and disabled passwords too. With access exclusively through
cryptographic keys, password-guessing attacks become impossible, not just unlikely.

Measured effect: after the change, the counter dropped from 8450 to ~114 attempts
between sessions.

---

## 10. What the other departments can do

| Department | Available now | What you get |
|---|---|---|
| **Backend** | MySQL + credentials, Node.js + PM2, continuous deployment | Deployment within 24h of a push to main |
| **Frontend** | API at https://sera.bytestorm.site/api/v1/ | NGINX serves the React build once you deliver it |
| **Hardware** | Endpoints at https://sera.bytestorm.site/api/v1/ + WebSocket /ws | Support on connection |
| **Entrepreneurship** | Live HTTPS server for demos | Stable server for the final demo |

**IMPORTANT - HTTPS migration:** the base URL is now **https://sera.bytestorm.site**.
Frontend and Hardware must use https://, not http:// - otherwise browsers block the
requests (mixed content).

**Workflow:** you do not need SSH access. You deliver on GitHub, Cloud pulls and
publishes on the server.

---

## 11. Repository structure

```
cloud/
+-- README.md                    <- this file
+-- JURNAL.md                    <- configuration history (log)
+-- RAPORT-saptamana-1.md
+-- .gitignore
```

---

## 12. Resources

- [AlmaLinux](https://wiki.almalinux.org/) - [NGINX](https://nginx.org/en/docs/) -
  [MySQL](https://dev.mysql.com/doc/) - [Certbot](https://certbot.eff.org/) -
  [PM2](https://pm2.keymetrics.io/)
- Internal installation guide - provided separately by the coordinator
