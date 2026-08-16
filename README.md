# Task Deployment - Osama 0xNETA

## Overview
Deployed the Cachet status-page application on a vanilla Ubuntu 24.04 LTS host,
following the architecture blueprint: Nginx -> PHP-FPM 8.3 (Cachet app layer) ->
MySQL / Redis, with RabbitMQ as an internal-only messaging gateway exposed
through a reverse proxy at `/rabbitmq`.

**Live environment:** http://165.227.110.238/
**RabbitMQ (via proxy only):** http://165.227.110.238/rabbitmq/

## Package Acquisition
- Core tools: `curl gnupg git unzip ca-certificates software-properties-common`
- PHP 8.3: `php8.3-fpm php8.3-mysql php8.3-redis php8.3-xml php8.3-mbstring
  php8.3-curl php8.3-bcmath php8.3-gd php8.3-zip php8.3-intl php8.3-cli`
- `nginx`, `mysql-server`, `redis-server`
- `erlang-nox` + `rabbitmq-server`, with `rabbitmq_management` plugin enabled
- App dependencies via Composer (`composer install --no-dev --optimize-autoloader`)

## Issues Identified in the Task Spec & How They Were Resolved

| Item | Issue | Resolution |
|---|---|---|
| `systemd-sysv-rpm` package | Not a real Ubuntu/apt package, `apt-get install` fails with "Unable to locate package" | Removed from the install command; `systemd-sysv` ships by default on 24.04 |
| `php artisan cachet:install` | Not a valid command in the current Cachet codebase (confirmed via the CLI's own "did you mean" suggestions) | Ran the equivalent setup manually: `php artisan migrate --force`, `php artisan cachet:make:user`, `php artisan cachet:assets`, `php artisan storage:link` |
| `proxy_strip_upstream_headers on;` | Not a valid Nginx directive, `nginx -t` fails with "unknown directive" | Used `proxy_buffering off;` on the `/rabbitmq` location to achieve the intended effect (no buffered upstream response data held server-side) |
| `fastcgi_buffer_max_chunk 512k;` | Not a valid Nginx directive, same failure mode | Used the real directive `fastcgi_max_temp_file_size 512k;` to cap on-disk buffering |
| `SESSION_DRIVER=redis_cluster` (.env baseline) | Not a supported Laravel/Cachet session driver | Set to `SESSION_DRIVER=redis` |
| `APP_URL=http://localhost` (.env baseline) | Breaks asset/link generation on a public-facing deployment | Set to the server's public IP |
| Filament UI broken (Alpine.js `isProcessing is not defined`, 404 on `index.css`) | Filament's frontend CSS/JS assets were not published automatically by `composer install` | Ran `php artisan filament:assets` followed by `php artisan optimize:clear` |

## Process Isolation (PHP-FPM)
- Created a **dedicated** PHP-FPM pool named `cachet` (`/etc/php/8.3/fpm/pool.d/cachet.conf`),
  running as `www-data:www-data`, listening only on a permission-locked Unix
  socket at `/run/php/cachet.sock` (mode `0660`, owner/group `www-data`), no
  TCP fallback.
- Nginx's `fastcgi_pass` points explicitly to `unix:/run/php/cachet.sock`
  (not the system-default `www` pool/socket), so the application runs in
  full isolation from any other PHP workload the box might host.
- **Verified**: temporarily disabled the default `www` pool entirely and
  confirmed the site continued serving `200 OK` using only the `cachet`
  pool's worker processes, confirming true isolation, not just a
  configuration label.
- Application files owned end-to-end by `www-data` (`chown -R www-data:www-data /var/www/cachet`).

## Nginx Configuration (`/etc/nginx/sites-available/cachet`)
- `server_name _` block serving `/var/www/cachet/public` as document root.
- PHP requests routed to the dedicated `cachet.sock` unix socket.
- `location /rabbitmq/` reverse-proxies to `http://127.0.0.1:15672/`,
  forwarding `Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`,
  with `proxy_buffering off;` to avoid buffering upstream response data.
- `fastcgi_max_temp_file_size 512k;` caps FastCGI on-disk buffer growth.
- `location ~ /\.ht` blocks access to dotfiles.

## Database
- MySQL database `cachet_db`, dedicated least-privilege user `cachet_user`
  scoped only to that schema (not root).
- Schema initialized via `php artisan migrate --force` (90+ migrations
  applied successfully).

## Redis
- Installed and enabled to persist across reboots (`systemctl enable --now redis-server`).
- Used by Cachet for sessions, cache, and queue driver, confirmed working
  via successful session cookie issuance on live requests.

## RabbitMQ & Firewall
- RabbitMQ deployed with the `rabbitmq_management` plugin enabled.
- UFW explicitly denies port `15672/tcp` (both IPv4 and IPv6) from any
  source, verified externally: a direct connection attempt from outside
  the network times out, while `/rabbitmq/` through Nginx returns `200 OK`.

```
$ sudo ufw status verbose
15672/tcp        DENY IN    Anywhere
15672/tcp (v6)   DENY IN    Anywhere (v6)
```

## Security Hardening
- `.env` file permissions restricted to `640`, owned by `www-data:www-data`
  (not world-readable/writable).
- MySQL app user scoped to a single schema, connecting only from `localhost`.

## Verification Performed
```bash
sudo systemctl status nginx php8.3-fpm mysql redis-server rabbitmq-server
sudo nginx -t
curl -I http://165.227.110.238/          # 200 OK
curl -I http://165.227.110.238/rabbitmq/ # 200 OK (via proxy)
curl -I http://165.227.110.238:15672/ --connect-timeout 5   # times out (blocked externally)
sudo ufw status verbose
```

## Known Follow-ups
- TLS/443 was allowed in UFW but no certificate was provisioned (no domain
  name was supplied for this test host), recommend Let's Encrypt/Certbot
  once a domain is pointed at the server.
- RabbitMQ is running with default `guest` credentials; should be rotated
  before any use beyond this test environment.
