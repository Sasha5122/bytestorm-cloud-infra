# Configuration log - ByteStorm Server

Every change on the server is recorded here, **at the moment it is made**.

**Why it matters:** a month from now you will not remember what you installed and with
which options. If the server has to be rebuilt from scratch, this log is the difference
between 30 minutes and two days. It is also proof of work for the evaluation.

**Format:** newest entry on **top**, most recent first.

**Server:** `169.58.12.143` - `vmi3438544.contaboserver.net` - AlmaLinux 9.8

---

## 2026-09-09 - HTTPS (Stage 5) + backend deploy + superadmin

**Stage:** 5 (HTTPS/SSL), plus continuous backend deployment

**What was done:**
1. HTTPS enabled on the domain sera.bytestorm.site
2. Successive backend deploys (git pull + PM2 restart)
3. Created the first superadmin through the official CLI
4. Enabled account approval by a superadmin

### HTTPS / SSL (Stage 5)
Project domain: sera.bytestorm.site (points to 169.58.12.143).
It is not echipa1.bytestorm.site (the example in the brief) - the real name is sera.

Domain check before Certbot:
getent hosts sera.bytestorm.site      # -> 169.58.12.143
curl -I http://sera.bytestorm.site    # -> 200, reaches the server

Install and certificate:
sudo dnf install certbot python3-certbot-nginx -y
sudo certbot --nginx -d sera.bytestorm.site

The automatic install failed initially ("Could not find a matching server block")
because server_name was `_`, not the domain name. Fixed:
- edited /etc/nginx/nginx.conf: server_name _ -> server_name sera.bytestorm.site
- sudo nginx -t (ok)
- sudo certbot install --cert-name sera.bytestorm.site -> deployed successfully

Auto-renewal:
sudo systemctl enable --now certbot-renew.timer
sudo certbot renew --dry-run    # -> all simulated renewals succeeded

Certificate valid until 2026-12-08, renews automatically.

Check:
https://sera.bytestorm.site -> Canopy page, with padlock (valid certificate)
curl https://sera.bytestorm.site/health -> status ok, database connected
API at https://sera.bytestorm.site/api/v1/

### Backend deployment (continuous)
Standard procedure on every new delivery from Backend:
cd ~/backend
git pull
pm2 restart bytestorm-backend
curl -i http://localhost:3000/health

Production .env created on the server (never committed to GitHub): MySQL credentials,
JWT_SECRET and other locally generated secrets, absolute paths, NODE_ENV=production.
Backend via NGINX reverse proxy: /api/, /health, /ws. Runs on localhost:3000,
port closed in the firewall.

### Superadmin (via CLI, not direct SQL)
Lesson: changing the role directly in SQL is NOT enough - the app keeps seeing the
user as 'utilizator'. The role must be granted through the application flow:
cd ~/backend
npm run admin:create -- --nume <NAME> --prenume <FIRSTNAME> --email <EMAIL> --parola <PASSWORD>
Verified with login: the role appears correctly in the response (data.user.rol = superadmin).

### Enabling account approval (.env)
At Backend's request: SUPERADMIN_APPROVAL_REQUIRED=false -> true, then pm2 restart.
Effect: new accounts are created inactive; login refused until a superadmin approves them.

---

## 2026-09-02 - Backend Node.js deployment (Stage 3.4-3.5)

**Stage:** 3.4, 3.5

**What was done:**
- Cloned the backend code from GitHub (private repo, authentication with a Personal Access Token)
- Installed dependencies with npm ci (433 packages)
- Generated the production secrets (JWT_SECRET and others)
- Created the production .env on the server (NOT committed to GitHub)
- Started the backend with PM2, configured NGINX as a reverse proxy
- API live and reachable from outside

**Main commands:**
sudo dnf install git -y
git clone https://github.com/Bytestorm0726/backend.git
cd backend
npm ci
node -e "console.log(require('crypto').randomBytes(64).toString('hex'))"
nano .env
pm2 start server.js --name bytestorm-backend
pm2 save

**.env configuration - key variables:**
- DB_HOST=localhost, DB_USER=sera_user, DB_NAME=sera_db
- NODE_ENV=production
- JWT_SECRET and the other secrets generated locally
- Absolute paths for AVATAR_ROOT, GDPR_EXPORT_ROOT, GDPR_LEDGER_ROOT
- Email disabled, but EMAIL_ACTION_SECRET requires at least 32 characters

**NGINX configuration (/etc/nginx/default.d/backend.conf):**
Reverse proxy for /api/, /health and /ws (hardware WebSocket).
Backend listens on localhost:3000; port 3000 stays closed in the firewall.

**Check:**
curl http://169.58.12.143/health -> status ok, database connected
Reachable from the browser via the public address too.

**Problems encountered:**
1. Git was not installed on the server (minimal image). Installed with dnf.
2. Private repo - cloning asks for a token (Personal Access Token, scope repo).
3. Production guards in series: GDPR_LEDGER_ROOT missing, unsafe demo account in
   the DB (ion@bytestorm.ro, deleted), EMAIL_ACTION_SECRET at least 32 characters.
   Resolved one by one - the code clearly states what is missing.
4. Restart loop under PM2 (EADDRINUSE): a leftover node process from a manual start
   was holding port 3000. Found with ss -tlnp | grep 3000, killed with kill -9.
5. Temporary 404 at /health - tested before the NGINX reload picked up the config.

**Impact for the team:**
API live at http://169.58.12.143/api/v1/. Hardware can send data (WebSocket on /ws).
Frontend can connect. The first real account is created via registration.

---

## 2026-08-03 - Additional MySQL users: Frontend and Hardware

**What was done:**
Created two additional MySQL users on the sera_db database, at the departments'
request, with differentiated privileges:

* sera_front (Frontend) - read only (SELECT)
* sera_hard (Hardware) - full access (ALL PRIVILEGES)

**Commands (as MySQL root):**
CREATE USER 'sera_front'@'localhost' IDENTIFIED BY '<password>';
GRANT SELECT ON sera_db.* TO 'sera_front'@'localhost';
CREATE USER 'sera_hard'@'localhost' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON sera_db.* TO 'sera_hard'@'localhost';
FLUSH PRIVILEGES;

**Check:**
SHOW GRANTS for sera_front -> only SELECT on sera_db (read-only).
SHOW GRANTS for sera_hard -> ALL PRIVILEGES on sera_db.
Both users tested with a direct connection; passwords confirmed.
The three users on sera_db: sera_user (Backend, full), sera_front
(Frontend, read-only), sera_hard (Hardware, full).

**Decisions:**
* All users are @localhost - access only from the server.
* Frontend got SELECT only: it displays data, does not modify it.
The restriction applies automatically in phpMyAdmin too.
* Each user is limited to sera_db, with no access to other databases.

**Problems encountered:**
When creating sera_front, the password was initially set to the placeholder text
instead of the real password. Fixed with ALTER USER - the user does not need to be
dropped and recreated, only the password is changed.

**Impact for the team:**
Frontend and Hardware can access the database, from code and through phpMyAdmin,
each with the assigned privileges. Credentials shared over a private channel.

---

## 2026-07-24 - Full reboot test

**What was done:**
Controlled reboot of the server to verify that all services come back automatically,
with no manual intervention.

**Commands:**
```bash
sudo reboot
# ~90 seconds
ssh admin@169.58.12.143
sudo systemctl status nginx mysqld php-fpm --no-pager
pm2 list
curl http://localhost:3000
```

**Check:**
All services back to `active (running)`. The PM2 test app `online`.
Landing page and phpMyAdmin reachable from the browser.

**Why it was done now:**
"Configured to start at boot" and "actually starts" are different things. The test
was done at a time when nobody depends on the server; a problem discovered a month
later, during active development, would have cost much more.

**Problems encountered:** none. All services came back correctly.

---

## 2026-07-24 - Stage 3.1-3.3: Node.js, PM2 and Express test

**What was done:**
* Installed Node.js 22 LTS and npm
* Installed PM2 globally
* Configured PM2 to auto-start at boot
* Tested the full chain with a minimal Express app

**Commands:**
```bash
# Node.js
dnf module list nodejs
sudo dnf module enable nodejs:22 -y
sudo dnf install nodejs -y
sudo dnf install npm -y
node -v      # v22.23.1
npm -v       # 10.9.8

# PM2
sudo npm install -g pm2
pm2 -v       # 7.0.3

# Express test
mkdir -p ~/test-backend && cd ~/test-backend
npm init -y
npm install express
nano index.js
pm2 start index.js --name test-backend
pm2 list
curl http://localhost:3000

# Auto-start at boot
pm2 startup
sudo env PATH=$PATH:/usr/bin /usr/local/lib/node_modules/pm2/bin/pm2 startup systemd -u admin --hp /home/admin
pm2 save
```

**Check:**
`pm2 list` -> `test-backend` with status `online`.
`curl http://localhost:3000` -> "Backend test OK - server is working!"
The `pm2-admin` service created in systemd.

**Why version 22:**
Even-numbered Node.js versions are LTS. 18 is near end of support, 20 is older, 24 is
too new (some npm packages do not support it fully yet). 22 is the balance between
maturity and long-term support.

**Why PM2:**
An app started with `node index.js` dies when the SSH session closes, does not restart
if it crashes, and does not start after a reboot. PM2 solves all three. On a 24/7
server it is mandatory.

**Problems encountered:**
`npm` was not included in the `nodejs` package on AlmaLinux - it had to be installed
separately with `dnf install npm`.

**Impact for the team:**
The backend infrastructure is ready. When Backend delivers code on GitHub, publishing
takes ~15 minutes. The Express test confirms that any later problem will be in the app
code, not in the server configuration.

---

## 2026-07-23 - Stage 2.5: phpMyAdmin

**What was done:**
* Installed PHP 8.2 + PHP-FPM + the required extensions
* Added the EPEL repository (phpMyAdmin is not in the official AlmaLinux repos)
* Installed phpMyAdmin 5.2.3
* Configured NGINX to serve it at an unusual path

**Commands:**
```bash
# PHP
dnf module list php
sudo dnf module enable php:8.2 -y
sudo dnf install php php-fpm php-mysqlnd php-json php-mbstring php-zip php-gd php-curl php-xml -y
php -v       # 8.2.32
sudo systemctl enable --now php-fpm

# phpMyAdmin
sudo dnf install epel-release -y
sudo dnf install phpmyadmin -y

# NGINX configuration
grep -E "^listen" /etc/php-fpm.d/www.conf     # socket: /run/php-fpm/www.sock
sudo nano /etc/nginx/default.d/phpmyadmin.conf
sudo nginx -t
sudo systemctl reload nginx
sudo setsebool -P httpd_can_network_connect 1
```

**Contents of `/etc/nginx/default.d/phpmyadmin.conf`:**
```nginx
location /db-sera-2026 {
    alias /usr/share/phpMyAdmin;
    index index.php;

    location ~ ^/db-sera-2026(/.*\.php)$ {
        alias /usr/share/phpMyAdmin;
        fastcgi_pass unix:/run/php-fpm/www.sock;
        fastcgi_param SCRIPT_FILENAME /usr/share/phpMyAdmin$1;
        include fastcgi_params;
    }
}
```

**Check:**
`http://169.58.12.143/db-sera-2026` -> login page.
Login with `sera_user` (not root) -> only `sera_db` visible.
`http://169.58.12.143` -> the Canopy landing page still works.

**Decision: the path `/db-sera-2026` instead of `/phpmyadmin`**
phpMyAdmin at the default path is one of the most scanned targets on the internet.
An unusual path is not real security, but it filters out all automated traffic with
zero effort. The address is not posted anywhere public.

Browser access was allowed deliberately during development: the DB holds only test
data, and restricting it (SSH tunnel) would have made the Cloud lead a middleman for
every check by the Backend team. Once the DB holds real data, access will be tightened.

**Problems encountered:**
1. **`location` directive is not allowed here.** The first config was placed in
`/etc/nginx/conf.d/`, which is included **outside** the `server` block in `nginx.conf`.
A `location` block cannot stand on its own. Fixed by moving the file to
`/etc/nginx/default.d/`, which is included **inside** the `server` block.
2. **`alias` vs `root`.** Because the path name (`/db-sera-2026`) does not match the
folder name (`/usr/share/phpMyAdmin`), `alias` was needed instead of `root`. `root`
appends the requested path; `alias` replaces it.
3. **SELinux blocks the NGINX -> PHP-FPM connection.** Fixed with
`setsebool -P httpd_can_network_connect 1`.

**Impact for the team:**
Backend can visually inspect the database contents, without SSH access to the server.

---

## 2026-07-23 - Stage 2.1-2.4, 2.6: MySQL

**What was done:**
* Installed and started MySQL Server 8.0.46
* Ran the hardening script
* Created the `sera_db` database
* Created the dedicated `sera_user` with limited privileges
* Shared the credentials with the Backend team

**Commands (system):**
```bash
sudo dnf list mysql-server
sudo dnf install mysql-server -y
sudo systemctl enable --now mysqld
sudo systemctl status mysqld
sudo mysql_secure_installation
```

**Answers to `mysql_secure_installation`:**
```
VALIDATE PASSWORD COMPONENT  -> n
New password                 -> (MySQL root password, stored offline)
Remove anonymous users       -> y
Disallow root login remotely -> y
Remove test database         -> y
Reload privilege tables      -> y
```

**Commands (SQL):**
```sql
CREATE DATABASE sera_db CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER 'sera_user'@'localhost' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON sera_db.* TO 'sera_user'@'localhost';
FLUSH PRIVILEGES;
SHOW GRANTS FOR 'sera_user'@'localhost';
```

**Check:**
As root, `SHOW DATABASES` -> 5 databases (including the system ones).
As `sera_user`, `SHOW DATABASES` -> only `information_schema`, `performance_schema`,
`sera_db`. The `mysql` and `sys` databases are not visible - the restriction works.

`SHOW GRANTS` confirms: `ALL PRIVILEGES ON sera_db.*`, not `*.*`.

**Decisions:**
* **`utf8mb4` / `utf8mb4_unicode_ci`** - the only encoding that fully supports Romanian
diacritics and any Unicode character. A database encoding is hard to change once it
holds data; set it correctly at creation.
* **`@'localhost'`** - the user can only connect from the server, not remotely. A second
barrier on top of port 3306 being closed in the firewall.
* **One user, not one per person** - `sera_user` is the application's identity, not a
person's. There is a single backend running, hence a single connection. Separate users
would be justified for different apps or separate environments (dev/prod), not for team
members.
* **Port 3306 stays closed** - the backend runs on the same server and connects via
`localhost`. The traffic never leaves the server.

**Credentials shared with the Backend team (2.6):**
```
DB_HOST=localhost
DB_PORT=3306
DB_NAME=sera_db
DB_USER=sera_user
DB_PASSWORD=<private channel>
```

With the explicit note that `.env` is never committed to GitHub and that a
`.env.example` with empty values is committed instead.

**Problems encountered:** none during configuration.

**Impact for the team:**
Backend can start development. The DB is empty - they create the tables from code.

---

## 2026-07-22 - "Canopy" landing page deployment

**Task:** Publish the static presentation page (build 1.0.0 from Entrepreneurship)

**What was done:**
* Transferred the landing page files from the local machine to the server
* Configured NGINX to serve the page from `/var/www/canopy`
* Replaced the default AlmaLinux page

**Commands (local machine, PowerShell):**
```powershell
scp -r C:\...\canopy-landing-1.0.0\* admin@169.58.12.143:/tmp/canopy-upload/
```

**Commands (server):**
```bash
# Prepare folders
sudo mkdir -p /var/www/canopy
mkdir -p /tmp/canopy-upload

# (scp transfer from the laptop into /tmp/canopy-upload)

# Move to the final location
sudo mv /tmp/canopy-upload/* /var/www/canopy/

# Owner and permissions
sudo chown -R nginx:nginx /var/www/canopy
sudo chmod -R 755 /var/www/canopy

# SELinux context
sudo restorecon -Rv /var/www/canopy

# NGINX configuration
sudo cp /etc/nginx/nginx.conf /etc/nginx/nginx.conf.backup
sudo nano /etc/nginx/nginx.conf
#   -> in the active server block (listen 80, no #):
#     root  /usr/share/nginx/html;  ->  root  /var/www/canopy;
sudo nginx -t
sudo systemctl reload nginx
```

**Check:**
`curl -s http://localhost/ | head -20` -> shows `<title>Canopy`.
Opening `http://169.58.12.143` -> landing page visible (after clearing the browser cache).

**Problems encountered:**
1. **SELinux blocked the files.** Copied from `/tmp`, they had the `user_tmp_t` label,
which NGINX is not allowed to serve (it would return 403). Fixed with `restorecon`,
which relabeled them to `httpd_sys_content_t`.
2. **The `assets` folder had `drwx------` permissions** (owner only). Fixed by
`chmod -R 755`.
3. **The browser showed the old AlmaLinux page after reload.** Cache. Confirmed with
`curl` on the server (it served Canopy correctly) and verified in an incognito window.

**Impact for the team:**
The Entrepreneurship department has a live page for demos at `http://169.58.12.143`.

---

## 2026-07-19 - Closing Week 1

**Stage 1 of the department brief is complete.**

Delivered:
* Server publicly reachable at `http://169.58.12.143`
* Secure SSH access, keys only
* Active firewall, only ports 22, 80, 443
* NGINX installed and working

Scheduled for Week 2:
* Stage 2 - MySQL, phpMyAdmin, database and dedicated user
* Stage 3 - Node.js, PM2, backend publishing

> The brief places Stage 2 in the "Week 1-2" range. Execution is on schedule.

Stages 4 and 5 (frontend served by NGINX, HTTPS) depend on code delivered by the other
departments and on domain registration.

---

## 2026-07-19 - Stage 1.5: NGINX

**What was done:**
* Installed NGINX from the official AlmaLinux repo
* Enabled auto-start at boot
* Verified browser access

**Commands:**
```bash
sudo dnf install nginx -y
sudo systemctl enable --now nginx
sudo systemctl status nginx
```

**Check:**
`systemctl status nginx` -> `active (running)`.
`http://169.58.12.143` shows the default AlmaLinux Web Server Test Page.

**Installed packages:** `nginx 1.20.1-28.el9_8.4`, `nginx-core`, `nginx-filesystem`,
`almalinux-logos-httpd`

**Problems encountered:** none.

**Impact for the team:**
The server is publicly reachable. Stage 1 complete. The IP was shared with the team.

---

## 2026-07-19 - Stage 1.4: Firewall

**What was done:**
* Found that `firewalld` **was not installed** in the Contabo image - the server was
running with no active firewall
* Installed and enabled auto-start at boot
* Opened ports 22 (SSH), 80 (HTTP), 443 (HTTPS)
* Removed the `cockpit` service (port 9090) - an unused web administration interface

**Commands:**
```bash
sudo dnf list installed | grep -E "firewalld|nftables|iptables"
sudo dnf install firewalld -y
sudo systemctl enable --now firewalld
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-service=https
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

**Check:**
```
public (active)
  interfaces: eth0
  services: dhcpv6-client http https ssh
  ports:
```

Ports 3306 (MySQL) and 8080 (Node.js) are not and will not be opened - communication
with them happens only locally, via `localhost`.

**Problems encountered:**
The first run of `firewall-cmd` returned `command not found`. The Contabo image is
minimal and does not include firewalld. Only `iptables-libs` was present (a library,
not an active firewall), so there was no conflict on install.

**Impact for the team:**
Attack surface reduced to three ports. No visible change for the other departments.

---

## 2026-07-19 - Stage 1.3: SSH access hardening

**What was done:**
* Generated an ed25519 key pair on the workstation
* Installed the public key on the server for the `admin` user
* Kept a backup of the private key offline
* Disabled direct `root` login
* Fully disabled password authentication

**Commands (local machine, PowerShell):**
```powershell
ssh-keygen -t ed25519
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh admin@169.58.12.143 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**Commands (server):**
```bash
sudo ls -la /etc/ssh/sshd_config.d/
sudo cat /etc/ssh/sshd_config.d/50-cloud-init.conf
sudo dnf install nano -y
sudo nano /etc/ssh/sshd_config.d/10-hardening.conf
sudo sshd -t
sudo systemctl reload sshd
sudo sshd -T | grep -E "permitrootlogin|passwordauthentication"
```

**Contents of `/etc/ssh/sshd_config.d/10-hardening.conf`:**
```
PermitRootLogin no
PasswordAuthentication no
```

**Check:**
```
permitrootlogin no
passwordauthentication no
```

Tests run from a new session, keeping the old session open as a fallback:
* `ssh root@169.58.12.143` -> `Permission denied (publickey,...)` OK
* `ssh -o PubkeyAuthentication=no admin@...` -> `Permission denied` OK
* `ssh admin@169.58.12.143` -> direct login, no password OK

**Problems encountered:**
1. **A file overriding the configuration.** The Contabo image contains
`/etc/ssh/sshd_config.d/50-cloud-init.conf` with a single line:
`PasswordAuthentication yes`. Files in this folder take priority over
`/etc/ssh/sshd_config`, and the first value found wins. Editing the main file would
have had no effect - the problem would have gone unnoticed. **Fix:** a new file with a
`10-` prefix, read before `50-`. The cloud-init file was not modified, since it is
auto-generated and may be rewritten.
2. **`nano` was not installed.** The minimal image does not include it. Installed with `dnf`.
3. **`sudo cat .../*.conf` failed** with `No such file or directory`. The `*` is expanded
by the shell before `sudo` starts, and the shell ran as `admin`, with no read permission
on the folder. Fixed by specifying the exact names.

**Impact for the team:**
Server access is exclusively through SSH keys. Anyone who needs access sends their
public key to the Cloud lead.

---

## 2026-07-19 - Stage 1.2: System update and non-root user

**What was done:**
* Checked the server state on handover
* Updated all system packages
* Rebooted to activate the new kernel
* Created the `admin` user with administrative rights

**Commands:**
```bash
hostnamectl
last -5
systemctl list-units --type=service --state=running
cat ~/.bash_history | tail -30

dnf update -y
dnf needs-restarting -r
reboot

adduser admin
passwd admin
usermod -aG wheel admin
id admin
```

**Check:**
```
uid=1001(admin) gid=1001(admin) groups=1001(admin),10(wheel)
```

`sudo whoami` -> `root`, tested from a separate session with the root session kept open
as a fallback.

**Update result:**
24 packages, including kernel and openssl.
Kernel: `5.14.0-687.24.1` -> `5.14.0-687.26.1`.
`dnf needs-restarting -r` flagged a required reboot. Done; server back online in ~90 seconds.

**Server state on handover:**
`last -5` shows a single previous login (the coordinator, 13 July, duration 00:00).
`bash_history` contains only `ls` and `exit`. No application service installed.
**Conclusion: clean server, configuration from scratch.**

**Problems encountered:** none.

**Impact for the team:**
The system is up to date. From now on, work is done exclusively as `admin`, not `root`.

---

## 2026-07-19 - Stage 1.1: Server handover and first SSH connection

**What was done:**
* Received the access details from the coordinator (IP, user `root`, password)
* First successful SSH connection
* Checked the operating system and resources

**Commands:**
```powershell
ssh root@169.58.12.143
```
```bash
hostnamectl
whoami
df -h
free -h
```

**Check:**
Prompt `[root@vmi3438544 ~]#` shown. `hostnamectl` confirms AlmaLinux 9.8
(Olive Jaguar), kernel 5.14.0-687.24.1, KVM virtualization.

**Problems encountered:**
1. **`Connection closed by 169.58.12.143 port 22`** on the first attempts. Diagnosed
with `ssh -v`. The log confirmed the server accepts password authentication
(`Authentications that can continue: publickey,...,password`), so the problem was not
in the server configuration. The cause turned out to be the password entry. Fixed by
re-copying the password from the original source.
2. **`Could not resolve hostname`** - caused by including the text `IP` in the command
(`ssh root@IP169.58.12.143`). The correct command contains only the numeric address.

**Security observation:**
The login banner reports **8450 failed authentication attempts** since the last
successful login, from various IP addresses. These are automated scans trying the
`root` user with common passwords - normal for any public server. This is the direct
justification for the measures in Stage 1.3.

**Impact for the team:**
The server is reachable. Nothing can be published yet - NGINX, MySQL and Node.js are
not installed.

---

<!-- New entries go ABOVE this line -->

---

## Template for new entries

```markdown
## YYYY-MM-DD - [Short title]
**Stage:** X.Y

**What was done:**
- ...

**Commands:**
```bash
command 1
```

**Check:** how I confirmed it works

**Problems encountered:** what went wrong and how it was fixed (or "none")

**Impact for the team:** what gets unblocked / what the others need to know
```
