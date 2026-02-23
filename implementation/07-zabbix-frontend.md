# Phase 7 — Zabbix Frontend (Nginx + PHP-FPM)

> **Objective:** Deploy and configure the Zabbix web frontend at both sites using Nginx and PHP-FPM, served behind KEMP LoadMaster Virtual Services configured in Phase 5.

---

## Prerequisites

| Requirement | Details |
|-------------|---------|
| Phase 6 complete | Zabbix Server HA cluster running (one active, one standby) |
| Packages installed | `zabbix-web-pgsql`, `zabbix-nginx-conf`, `php-fpm`, `nginx` installed on `{{FRONTEND_A_IP}}` and `{{FRONTEND_B_IP}}` |
| KEMP Virtual Services | Frontend HTTPS (port 443) and HTTP redirect (port 80) configured and healthy |
| Database accessible | `{{VIP_DB_A}}` and `{{VIP_DB_B}}` routing to Patroni primary |
| DNS record | `{{FQDN_FRONTEND}}` resolving to `{{VIP_FRONTEND_A}}` (or GSLB if applicable) |

---

## Step 1 — Configure PHP-FPM

Edit the Zabbix PHP-FPM pool configuration on **each frontend VM**.

**File:** `/etc/php-fpm.d/zabbix.conf`

```bash
sudo vi /etc/php-fpm.d/zabbix.conf
```

Set or verify the following parameters:

```ini
[zabbix]
user = nginx
group = nginx

listen = /run/php-fpm/zabbix.sock
listen.acl_users = nginx
listen.allowed_clients = 127.0.0.1

pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35

php_value[session.save_handler] = files
php_value[session.save_path]    = /var/lib/php/session

; Timezone — set to your local timezone or UTC
php_value[date.timezone] = UTC

; Zabbix-recommended PHP settings
php_value[max_execution_time] = 300
php_value[memory_limit] = 128M
php_value[post_max_size] = 16M
php_value[upload_max_filesize] = 2M
php_value[max_input_time] = 300
php_value[max_input_vars] = 10000
```

> **Important:** The default Zabbix PHP-FPM pool runs as `apache:apache`. Since we use Nginx, change `user` and `group` to `nginx`. If this is not changed, Nginx will receive "Permission denied" errors when connecting to the PHP-FPM socket.

Add session security hardening:

```bash
# Session security hardening
echo 'php_value[session.cookie_secure] = On' >> /etc/php-fpm.d/zabbix.conf
echo 'php_value[session.cookie_httponly] = On' >> /etc/php-fpm.d/zabbix.conf
echo 'php_value[session.cookie_samesite] = Strict' >> /etc/php-fpm.d/zabbix.conf
echo 'php_value[session.use_strict_mode] = 1' >> /etc/php-fpm.d/zabbix.conf
```

Ensure the session directory exists and has correct ownership:

```bash
sudo mkdir -p /var/lib/php/session
sudo chown -R nginx:nginx /var/lib/php/session
sudo chmod 770 /var/lib/php/session
```

Repeat on `{{FRONTEND_B_IP}}`.

---

## Step 2 — Configure Nginx

Edit the Zabbix Nginx configuration on **each frontend VM**.

**File:** `/etc/nginx/conf.d/zabbix.conf`

```bash
sudo vi /etc/nginx/conf.d/zabbix.conf
```

Uncomment and set the `listen` and `server_name` directives:

```nginx
server {
    listen          80;
    server_name     {{FQDN_FRONTEND}};

    server_tokens off;

    root    /usr/share/zabbix;

    index   index.php;

    # Security headers
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "SAMEORIGIN" always;
    add_header X-XSS-Protection "1; mode=block" always;
    add_header Referrer-Policy "strict-origin-when-cross-origin" always;

    client_max_body_size 16M;

    location = /favicon.ico {
        log_not_found   off;
    }

    location / {
        try_files       $uri $uri/ =404;
    }

    location /assets {
        access_log      off;
        expires         10d;
    }

    location ~ /\.ht {
        deny            all;
    }

    location ~ /(api\/|conf[^\.]|include|locale|vendor) {
        deny            all;
        return          404;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_pass    unix:/run/php-fpm/zabbix.sock;
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_index   index.php;

        fastcgi_param   DOCUMENT_ROOT   /usr/share/zabbix;
        fastcgi_param   SCRIPT_FILENAME /usr/share/zabbix$fastcgi_script_name;
        fastcgi_param   PATH_TRANSLATED /usr/share/zabbix$fastcgi_script_name;

        include fastcgi_params;
        fastcgi_param   QUERY_STRING    $query_string;
        fastcgi_param   REQUEST_METHOD  $request_method;
        fastcgi_param   CONTENT_TYPE    $content_type;
        fastcgi_param   CONTENT_LENGTH  $content_length;

        fastcgi_intercept_errors        on;
        fastcgi_ignore_client_abort     off;
        fastcgi_connect_timeout         60;
        fastcgi_send_timeout            180;
        fastcgi_read_timeout            180;
        fastcgi_buffer_size             128k;
        fastcgi_buffers                 4 256k;
        fastcgi_busy_buffers_size       256k;
        fastcgi_temp_file_write_size    256k;
    }
}
```

> **Note:** Nginx listens on port 80 (HTTP). SSL termination is handled by KEMP LoadMaster, which connects to the backend over HTTP. Do not configure SSL in Nginx unless you require end-to-end encryption (SSL re-encryption on KEMP).

Remove or rename the default Nginx configuration to avoid port conflicts:

```bash
sudo mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.disabled 2>/dev/null || true
```

Test the Nginx configuration:

```bash
sudo nginx -t
```

Expected output:

```
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
```

Repeat on `{{FRONTEND_B_IP}}`.

---

## Step 3 — Configure Zabbix Frontend (Site A)

Edit the Zabbix web configuration on `{{FRONTEND_A_IP}}`.

**File:** `/etc/zabbix/web/zabbix.conf.php`

```bash
sudo vi /etc/zabbix/web/zabbix.conf.php
```

```php
<?php
// Zabbix GUI configuration file.

$DB['TYPE']                     = 'POSTGRESQL';
$DB['SERVER']                   = '{{VIP_DB_A}}';
$DB['PORT']                     = '{{DB_PORT}}';
$DB['DATABASE']                 = 'zabbix';
$DB['USER']                     = 'zabbix';
$DB['PASSWORD']                 = '{{DB_ZABBIX_PASSWORD}}';

// Schema name. Used for PostgreSQL.
$DB['SCHEMA']                   = '';

// Used for TLS connection to the database.
// If PostgreSQL SSL is enabled (Phase 04), set to true and populate paths below
$DB['ENCRYPTION']               = false;
$DB['KEY_FILE']                 = '';
$DB['CERT_FILE']                = '';
$DB['CA_FILE']                  = '';
$DB['VERIFY_HOST']              = false;
$DB['CIPHER_LIST']              = '';

// Vault configuration (uncomment if using HashiCorp Vault).
// $DB['VAULT']                 = '';
// $DB['VAULT_URL']             = '';
// $DB['VAULT_DB_PATH']         = '';
// $DB['VAULT_TOKEN']           = '';

// Use the following to enable SAML authentication.
// $SSO['SETTINGS']             = [];

// HA: Do NOT set ZBX_SERVER or ZBX_SERVER_PORT.
// The frontend auto-detects the active HA node from the database.
// Uncomment only for non-HA single-server deployments:
// $ZBX_SERVER                  = 'localhost';
// $ZBX_SERVER_PORT             = '10051';
// $ZBX_SERVER_NAME             = '';

$IMAGE_FORMAT_DEFAULT           = IMAGE_FORMAT_PNG;
```

> **Critical:** In an HA deployment, do NOT set `$ZBX_SERVER` or `$ZBX_SERVER_PORT`. The frontend queries the `ha_node` table in the database to discover which Zabbix Server node is currently active. Setting these values would bypass HA-aware server discovery and hardcode a single server address.

Set correct file permissions:

```bash
sudo chown root:nginx /etc/zabbix/web/zabbix.conf.php
sudo chmod 640 /etc/zabbix/web/zabbix.conf.php
```

---

## Step 4 — Configure Zabbix Frontend (Site B)

Edit the Zabbix web configuration on `{{FRONTEND_B_IP}}`.

**File:** `/etc/zabbix/web/zabbix.conf.php`

```php
<?php
// Zabbix GUI configuration file.

$DB['TYPE']                     = 'POSTGRESQL';
$DB['SERVER']                   = '{{VIP_DB_B}}';
$DB['PORT']                     = '{{DB_PORT}}';
$DB['DATABASE']                 = 'zabbix';
$DB['USER']                     = 'zabbix';
$DB['PASSWORD']                 = '{{DB_ZABBIX_PASSWORD}}';

$DB['SCHEMA']                   = '';

// If PostgreSQL SSL is enabled (Phase 04), set to true and populate paths below
$DB['ENCRYPTION']               = false;
$DB['KEY_FILE']                 = '';
$DB['CERT_FILE']                = '';
$DB['CA_FILE']                  = '';
$DB['VERIFY_HOST']              = false;
$DB['CIPHER_LIST']              = '';

$IMAGE_FORMAT_DEFAULT           = IMAGE_FORMAT_PNG;
```

> **Note:** Site B's frontend connects to `{{VIP_DB_B}}`, the local KEMP Database VIP. This minimizes cross-site traffic for web UI operations. Both VIPs route to the same Patroni primary, ensuring data consistency.

Set correct file permissions:

```bash
sudo chown root:nginx /etc/zabbix/web/zabbix.conf.php
sudo chmod 640 /etc/zabbix/web/zabbix.conf.php
```

---

## Step 5 — Start Services

Start and enable PHP-FPM and Nginx on **both** frontend VMs.

**On `{{FRONTEND_A_IP}}`:**

```bash
sudo systemctl enable --now php-fpm
sudo systemctl enable --now nginx
```

**On `{{FRONTEND_B_IP}}`:**

```bash
sudo systemctl enable --now php-fpm
sudo systemctl enable --now nginx
```

Verify both services are running:

```bash
sudo systemctl status php-fpm
sudo systemctl status nginx
```

---

## Step 6 — Verify Local Access

Test the frontend directly on each VM (bypassing KEMP) to confirm Nginx and PHP-FPM are working.

**On `{{FRONTEND_A_IP}}`:**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost/
```

Expected: `200`

**On `{{FRONTEND_B_IP}}`:**

```bash
curl -s -o /dev/null -w "%{http_code}" http://localhost/
```

Expected: `200`

If the response is not 200, check the Nginx error log:

```bash
sudo tail -50 /var/log/nginx/error.log
```

---

## Step 7 — Verify Access Through KEMP VIP

Open a web browser and navigate to:

```
https://{{FQDN_FRONTEND}}
```

You should see the Zabbix login page served over HTTPS. Verify the following:

1. The SSL certificate is valid (no browser warnings)
2. The URL matches `{{FQDN_FRONTEND}}`
3. The login page renders correctly

Also verify from the command line:

```bash
# Through Site A VIP
curl -k -s -o /dev/null -w "%{http_code}" https://{{VIP_FRONTEND_A}}/

# Through Site B VIP
curl -k -s -o /dev/null -w "%{http_code}" https://{{VIP_FRONTEND_B}}/
```

Expected: `200` for both.

Verify the HTTP-to-HTTPS redirect:

```bash
curl -s -o /dev/null -w "%{http_code}" http://{{VIP_FRONTEND_A}}/
```

Expected: `301`

---

## Step 8 — Initial Web Setup Wizard

If this is a **first-time installation** (not a migration), the Zabbix setup wizard will appear when you first access the frontend.

Walk through the wizard:

| Step | Action |
|------|--------|
| 1. Welcome | Click **Next step** |
| 2. Check of prerequisites | All checks should show `OK`. Fix any failures before proceeding. |
| 3. Configure DB connection | Type: PostgreSQL, Host: `{{VIP_DB_A}}`, Port: `5432`, Database: `zabbix`, User: `zabbix`, Password: `{{DB_ZABBIX_PASSWORD}}` |
| 4. Settings | Zabbix server name: leave blank (HA mode), Default timezone: UTC or local, Default theme: Blue |
| 5. Pre-installation summary | Review all settings |
| 6. Install | Click **Finish** |

> **Note:** If `zabbix.conf.php` was already created in Step 3, the wizard may be skipped. This is expected behavior — the wizard only appears when the configuration file is absent or incomplete.

Default login credentials:

| Field | Value |
|-------|-------|
| Username | `Admin` |
| Password | `zabbix` |

**Change the default Admin password immediately** after first login:

1. Navigate to **Users > Users > Admin**
2. Click **Change password**
3. Set a strong password and save

---

## Step 9 — Set Global Macro {$ZABBIX.URL}

This macro is used by alert notifications, map URLs, and other features that need to generate links to the Zabbix frontend.

1. Log in to the Zabbix frontend as `Admin`
2. Navigate to **Administration > Macros**
3. Click **Add** to create a new global macro:

| Macro | Value | Type |
|-------|-------|------|
| `{$ZABBIX.URL}` | `{{ZABBIX_FRONTEND_URL}}` | Text |

4. Click **Update** to save

> **Note:** Use the public-facing URL (e.g., `https://zabbix.example.com`) — the one users and alert recipients will click on. Do not use an internal IP address.

---

## Verification Checkpoint

| # | Check | Expected Result | Command / Location |
|---|-------|-----------------|--------------------|
| 1 | PHP-FPM running — Site A | `active (running)` | `systemctl status php-fpm` on `{{FRONTEND_A_IP}}` |
| 2 | PHP-FPM running — Site B | `active (running)` | `systemctl status php-fpm` on `{{FRONTEND_B_IP}}` |
| 3 | Nginx running — Site A | `active (running)` | `systemctl status nginx` on `{{FRONTEND_A_IP}}` |
| 4 | Nginx running — Site B | `active (running)` | `systemctl status nginx` on `{{FRONTEND_B_IP}}` |
| 5 | Local access — Site A | HTTP 200 | `curl -s -o /dev/null -w "%{http_code}" http://localhost/` on `{{FRONTEND_A_IP}}` |
| 6 | Local access — Site B | HTTP 200 | `curl -s -o /dev/null -w "%{http_code}" http://localhost/` on `{{FRONTEND_B_IP}}` |
| 7 | Frontend via KEMP VIP | Login page visible at `https://{{FQDN_FRONTEND}}` | Browser |
| 8 | Both sites show same data | Dashboard content identical | Log in via both `{{VIP_FRONTEND_A}}` and `{{VIP_FRONTEND_B}}` |
| 9 | HA status in frontend | Active + Standby nodes visible | **Reports > System information** in Zabbix UI |
| 10 | {$ZABBIX.URL} macro set | `{{ZABBIX_FRONTEND_URL}}` | **Administration > Macros** |
| 11 | HTTP redirect working | 301 to HTTPS | `curl -I http://{{VIP_FRONTEND_A}}/` |
| 12 | Services enabled on boot | `enabled` | `systemctl is-enabled php-fpm nginx` on both nodes |

---

## Troubleshooting

### 502 Bad Gateway

**Symptoms:** Browser shows "502 Bad Gateway" when accessing the Zabbix frontend through Nginx.

**Resolution:**

1. Check if PHP-FPM is running:
   ```bash
   sudo systemctl status php-fpm
   ```
2. If PHP-FPM is stopped, start it:
   ```bash
   sudo systemctl start php-fpm
   ```
3. Verify the PHP-FPM socket exists:
   ```bash
   ls -la /run/php-fpm/zabbix.sock
   ```
4. If the socket is missing, check for startup errors:
   ```bash
   sudo journalctl -u php-fpm --no-pager -n 50
   ```
5. Verify the Nginx configuration points to the correct socket path:
   ```bash
   grep fastcgi_pass /etc/nginx/conf.d/zabbix.conf
   # Expected: fastcgi_pass unix:/run/php-fpm/zabbix.sock;
   ```
6. Check socket permissions — the `listen.acl_users` in the PHP-FPM pool must include `nginx`:
   ```bash
   grep listen.acl_users /etc/php-fpm.d/zabbix.conf
   # Expected: listen.acl_users = nginx
   ```
7. If the user/group was not changed from `apache` to `nginx`, fix it and restart:
   ```bash
   sudo systemctl restart php-fpm
   ```

### Database Connection Failed

**Symptoms:** Frontend shows "Error connecting to database" or the setup wizard cannot connect.

**Resolution:**

1. Verify the database VIP is reachable from the frontend VM:
   ```bash
   psql -h {{VIP_DB_A}} -p 5432 -U zabbix -d zabbix -c "SELECT 1;"
   ```
2. Check `zabbix.conf.php` has the correct database connection settings:
   ```bash
   sudo grep -E "SERVER|PORT|DATABASE|USER" /etc/zabbix/web/zabbix.conf.php
   ```
3. Verify the KEMP PostgreSQL Virtual Service is healthy:
   ```bash
   curl -k -s "https://bal:{{KEMP_ADMIN_PASSWORD}}@{{KEMP_A1_IP}}:8443/access/showvs?vs={{VIP_DB_A}}&port=5432&prot=tcp" | xmllint --format -
   ```
4. Ensure `pg_hba.conf` allows connections from the frontend IPs (`{{FRONTEND_A_IP}}` and `{{FRONTEND_B_IP}}`)
5. Test with the password explicitly:
   ```bash
   PGPASSWORD='{{DB_ZABBIX_PASSWORD}}' psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT 1;"
   ```

### Frontend Shows Wrong HA Node or "Zabbix server is not running"

**Symptoms:** The **Reports > System information** page shows "Zabbix server is not running" even though the server is active.

**Resolution:**

1. Verify `$ZBX_SERVER` and `$ZBX_SERVER_PORT` are **NOT** set in `zabbix.conf.php`:
   ```bash
   sudo grep -E "ZBX_SERVER|ZBX_SERVER_PORT" /etc/zabbix/web/zabbix.conf.php
   ```
   These lines should be commented out or absent.
2. Verify the frontend can query the `ha_node` table:
   ```bash
   psql -h {{VIP_DB_A}} -U zabbix -d zabbix -c "SELECT name, address, status FROM ha_node;"
   ```
3. Verify the active Zabbix server node has `status = 0` (active) in the `ha_node` table
4. Check that the frontend server can reach the active Zabbix server on port 10051:
   ```bash
   # From the frontend VM
   nc -zv {{ZBX_SERVER_A_IP}} 10051
   ```
5. If `$ZBX_SERVER` was accidentally set, comment it out and restart PHP-FPM:
   ```bash
   sudo systemctl restart php-fpm
   ```

### Nginx Configuration Test Fails

**Symptoms:** `nginx -t` reports syntax errors.

**Resolution:**

1. Check the error message for the specific file and line number
2. Common issues:
   - Missing semicolon at end of directive
   - Duplicate `listen` directives (check for `default.conf` conflict)
   - PHP-FPM socket path mismatch
3. Remove conflicting default configuration:
   ```bash
   sudo mv /etc/nginx/conf.d/default.conf /etc/nginx/conf.d/default.conf.disabled
   ```
4. Re-test:
   ```bash
   sudo nginx -t
   ```

### PHP Prerequisites Check Fails During Setup Wizard

**Symptoms:** The "Check of prerequisites" step shows red failures.

**Resolution:**

Common failing prerequisites and fixes:

| Prerequisite | Fix |
|-------------|-----|
| PHP timezone | Set `php_value[date.timezone]` in `/etc/php-fpm.d/zabbix.conf` |
| PHP memory_limit | Set `php_value[memory_limit] = 128M` in `/etc/php-fpm.d/zabbix.conf` |
| PHP max_execution_time | Set `php_value[max_execution_time] = 300` |
| PHP gd module | `sudo dnf install php-gd` (RHEL/Rocky) |
| PHP pgsql module | `sudo dnf install php-pgsql` (RHEL/Rocky) |
| PHP ldap module | `sudo dnf install php-ldap` (only if LDAP auth is needed) |

After installing modules or changing configuration, restart PHP-FPM:

```bash
sudo systemctl restart php-fpm
```
