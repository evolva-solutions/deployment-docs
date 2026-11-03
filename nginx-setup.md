# Nginx Setup on Ubuntu

Steps to install Nginx, serve a site, and reverse-proxy a Node.js app on Ubuntu. Commands assume you are logged in as a user who can run `sudo`.

## Before you start

Confirm the OS:

```bash
cat /etc/os-release
```

If a Node.js app will sit behind Nginx, start that app first and note the port it listens on. The [PM2 setup](pm2-setup-and-commands.md) uses port `3000` in the examples below.

## Install Nginx

```bash
sudo apt update
sudo apt install -y nginx
sudo systemctl enable --now nginx
```

Check the version and service:

```bash
nginx -v
sudo systemctl status nginx
```

Site files live in `/etc/nginx/sites-available/`. A symlink in `/etc/nginx/sites-enabled/` turns a site on. The package also enables a default site that answers on port `80`.

## Static site

Create the document root and a placeholder page:

```bash
sudo mkdir -p /var/www/example
sudo chown -R $USER:$USER /var/www/example
echo '<h1>example</h1>' > /var/www/example/index.html
```

Save this as `/etc/nginx/sites-available/example`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    root /var/www/example;
    index index.html;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

Enable it and drop the default site so it does not catch the same requests:

```bash
sudo ln -s /etc/nginx/sites-available/example /etc/nginx/sites-enabled/example
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Replace `example.com` with the real hostname. Point that name at this server before you request a certificate.

## Reverse proxy to a Node.js app

Use this when the app is already listening on localhost, for example a PM2 process on port `3000`. Nginx accepts public HTTP and forwards it to the app. The app stays bound to `127.0.0.1`.

Save this as `/etc/nginx/sites-available/app`:

```nginx
server {
    listen 80;
    listen [::]:80;
    server_name example.com www.example.com;

    location / {
        proxy_pass http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
    }
}
```

`Upgrade` and `Connection` keep WebSocket connections working. Remove those two lines if the app does not use them.

Enable the site:

```bash
sudo ln -s /etc/nginx/sites-available/app /etc/nginx/sites-enabled/app
sudo rm -f /etc/nginx/sites-enabled/default
sudo nginx -t
sudo systemctl reload nginx
```

Check that both sides are listening:

```bash
ss -lntp | grep -E ':80|:3000'
curl -I http://127.0.0.1
```

## Firewall

Allow HTTP and HTTPS, then confirm the rules:

```bash
sudo ufw allow 'Nginx Full'
sudo ufw status
```

`Nginx Full` opens ports `80` and `443`. `Nginx HTTP` opens only port `80`.

## TLS with Certbot

Install Certbot after the site answers on port `80` for the real hostname:

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d example.com -d www.example.com
```

Certbot edits the site file, listens on port `443`, and installs a renewal timer. Confirm renewal with a dry run:

```bash
sudo certbot renew --dry-run
sudo systemctl status certbot.timer
```

## Everyday commands

| Task | Command |
| --- | --- |
| Test configuration | `sudo nginx -t` |
| Reload after a config change | `sudo systemctl reload nginx` |
| Restart the service | `sudo systemctl restart nginx` |
| Stop | `sudo systemctl stop nginx` |
| Service status | `sudo systemctl status nginx` |
| Follow the error log | `sudo tail -f /var/log/nginx/error.log` |
| Follow the access log | `sudo tail -f /var/log/nginx/access.log` |

`nginx -t` must succeed before `reload`. A reload keeps existing connections. Use `restart` when a reload does not pick up the change.

## Useful checks

| Task | Command |
| --- | --- |
| Installed version | `nginx -v` |
| Full build info | `nginx -V` |
| Listening ports | `sudo ss -lntp \| grep nginx` |
| Which files are loaded | `sudo nginx -T` |
| Recent service logs | `sudo journalctl -u nginx -n 50 --no-pager` |
| Local response headers | `curl -I http://127.0.0.1` |

`nginx -T` prints the full configuration Nginx will use, including files pulled in from `sites-enabled`.

## Notes

- Keep application ports on localhost. Open ports `80` and `443` to the internet, and leave `3000` closed in the firewall.
- Run `sudo nginx -t` after every edit. Reload only when the test passes.
- One `server_name` should exist in only one active file. Two sites that claim the same name make Nginx pick one of them silently.
- Certificate files live under `/etc/letsencrypt/`. Do not copy them into the site directory.
