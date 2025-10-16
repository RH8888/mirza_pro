# Installation Guide

This guide walks through deploying the Mirza Pro Telegram bot panel on a production server. It summarizes the requirements inferred directly from the project source code and explains how to stand up the PHP application, database, Telegram webhook, and background tasks.

## 1. Choose a Hosting Environment

The bot processes incoming Telegram webhooks through `index.php` and schedules many background jobs from the `cronbot/` directory. That means the host must expose an HTTPS endpoint, allow outbound HTTP(S) calls, and support cron or CLI access to run scheduled scripts. A VPS or dedicated cloud instance is recommended so you can install system packages and manage crontab entries yourself. Shared hosting may work only if it offers:

- PHP 8.1 or newer with CLI access (vendor libraries such as `bacon/bacon-qr-code` require PHP 8.1+).【F:vendor/composer/installed.json†L1-L34】
- MySQL with the ability to create databases/users and enable both `mysqli` and `pdo_mysql` extensions (see `config.php`).【F:config.php†L2-L16】
- PHP extensions: cURL, GD, iconv/mbstring, fileinfo, openssl, and intl. The code checks for `curl` and `gd` explicitly and builds QR codes through `Endroid\QrCode`.【F:ibsng/Modules/IBSng.php†L25-L34】【F:vendor/endroid/qr-code/src/Writer/AbstractGdWriter.php†L30-L86】【F:function.php†L77-L121】
- Cron jobs or another scheduler capable of hitting the scripts in `cronbot/` at frequent intervals.【F:function.php†L1077-L1100】【F:cronbot/statusday.php†L1-L144】

If the shared host cannot satisfy these requirements, deploy to a VPS (Ubuntu 22.04+, Debian, etc.) where you can install the full stack.

## 2. Prepare the Server

1. **Install system packages** (example for Ubuntu with Nginx; adapt to Apache if preferred):
   ```bash
   sudo apt update && sudo apt install -y nginx mariadb-server php-fpm php-cli php-mysql php-curl php-xml php-mbstring php-gd php-intl php-zip php-bcmath php-redis unzip git
   ```
   Enable HTTPS with a real TLS certificate (Let’s Encrypt or your provider) so Telegram can deliver webhooks.

2. **Secure MySQL** and create a database/user:
   ```sql
   CREATE DATABASE mirza CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
   CREATE USER 'mirza_bot'@'localhost' IDENTIFIED BY 'strong_password';
   GRANT ALL PRIVILEGES ON mirza.* TO 'mirza_bot'@'localhost';
   FLUSH PRIVILEGES;
   ```

3. **Configure the web server** to serve the repository root (or the public subdirectory) via HTTPS. Point the document root to the folder that contains `index.php`. Make sure PHP-FPM is enabled and that the firewall (ufw, security groups, etc.) allows inbound TCP 443.

## 3. Deploy the Application Code

1. **Clone or upload the repository** into `/var/www/mirza_pro` (or another directory served by your web server):
   ```bash
   sudo mkdir -p /var/www/mirza_pro
   sudo chown $USER:$USER /var/www/mirza_pro
   git clone <your-fork-or-release-url> /var/www/mirza_pro
   ```

2. **Install PHP dependencies.** The repository already ships with a `vendor/` directory; if it ever goes missing run `composer install` from the project root to rebuild it.

3. **Set writable directories.** The application writes log files such as `error_log` and `log.txt` in the project root during runtime.【F:function.php†L36-L43】 Ensure the web server user (e.g., `www-data`) can write to these files:
   ```bash
   sudo chown -R www-data:www-data /var/www/mirza_pro
   sudo find /var/www/mirza_pro -type f -name "error_log" -o -name "log.txt" -exec chmod 664 {} +
   sudo find /var/www/mirza_pro -type d -exec chmod 775 {} +
   ```

## 4. Configure Application Secrets

1. **Edit `config.php`.** Replace the placeholders with your actual database credentials, Telegram bot token, admin numeric Telegram ID, public domain, and bot username.【F:config.php†L2-L16】
   ```php
   $dbname = 'mirza';
   $usernamedb = 'mirza_bot';
   $passworddb = 'strong_password';
   $APIKEY = '123456:ABC-YourTelegramBotToken';
   $adminnumber = '123456789';
   $domainhosts = 'bot.example.com';
   $usernamebot = 'YourBotHandle';
   ```
   The `$domainhosts` value is used in payment callbacks and cron URLs, so set it to the HTTPS domain that serves the bot.【F:function.php†L93-L121】【F:function.php†L1088-L1100】

2. **Optional panel flags.** `$new_marzban` toggles features for Marzban panel integrations and defaults to `true` in the template.【F:config.php†L15-L16】

## 5. Initialize the Database

Run the schema bootstrap script once. It creates and patches all required tables (users, settings, invoices, etc.) via PDO:
```bash
cd /var/www/mirza_pro
php table.php
```
The script checks `information_schema` and either creates tables or adds missing columns automatically.【F:table.php†L1-L200】 Execute it again after upgrading to ensure new fields are applied.

## 6. Configure Telegram Webhook

1. Ensure your domain uses HTTPS and resolves publicly; Telegram will reject self-signed certificates.
2. Register the webhook by calling:
   ```bash
   curl "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/setWebhook?url=https://bot.example.com/index.php"
   ```
   Telegram will then POST updates to `index.php`, which loads all runtime dependencies, enforces Telegram IP ranges, and handles users via the database and panels.【F:index.php†L1-L92】【F:index.php†L47-L52】
3. If you need to verify Telegram-only access, the code already validates inbound IP ranges through `checktelegramip()`.【F:function.php†L1060-L1075】

## 7. Schedule Background Jobs

The project expects frequent cron invocations to send reports, expire services, poll payment gateways, and manage panels. You can either:

- Call `activecron()` from a privileged CLI session to append the recommended cron entries automatically (requires shell access).【F:function.php†L1077-L1100】
- Or manually add cron jobs similar to:
  ```cron
  */15 * * * * curl -fsS https://bot.example.com/cronbot/statusday.php > /dev/null
  */5  * * * * curl -fsS https://bot.example.com/cronbot/payment_expire.php > /dev/null
  */1  * * * * curl -fsS https://bot.example.com/cronbot/sendmessage.php > /dev/null
  ```
  Review the scripts inside `cronbot/` (e.g., `statusday.php`) to understand their scheduling expectations and adjust frequencies accordingly.【F:cronbot/statusday.php†L1-L144】

If your host does not allow `crontab`, you must use an external scheduler (systemd timers, GitHub Actions, or third-party cron services) pointed at the same endpoints.

## 8. Configure Payment and Panel Webhooks

- The bot listens for JSON webhooks in `webhooks.php`. Requests must include an `X-Webhook-Secret` header whose base64-decoded value matches an administrator password stored in the `admin` table.【F:webhooks.php†L9-L20】
- Payment integrations such as NowPayments call back to `https://<domain>/payment/nowpayment.php`, so keep `$domainhosts` accurate.【F:function.php†L93-L121】
- Panel automations rely on the `panels.php` API wrapper; ensure your Marzban/x-ui credentials are stored through the admin UI after deployment.

## 9. Final Checklist

- Create the necessary admin records and configure channels/topics inside the bot UI. On the first Telegram interaction, `index.php` auto-creates user rows and reports new users to the configured channel.【F:index.php†L54-L91】
- Confirm outbound HTTPS access so the bot can reach Telegram, NowPayments, and panel APIs.【F:botapi.php†L3-L38】【F:function.php†L93-L144】
- Monitor `error_log`/`log.txt` and your web server logs for troubleshooting.【F:function.php†L36-L43】
- Backup the database regularly; the bot’s business state (users, invoices, reports) is stored in MySQL tables created by `table.php`.

Following these steps on a VPS or any host that satisfies the prerequisites will bring the Mirza Pro bot online with webhook handling, background automations, and payment notifications fully functional.
