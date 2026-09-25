# Hostinger VPS deployment

This repository is a static website. It needs no Node process or database in production; Nginx serves the HTML, CSS, JavaScript, images, and videos. The Nginx site config accounts for the homepage being named `Index.html`.

## 1. Prepare the VPS

Connect to the Ubuntu VPS (replace `root` if Hostinger gave you another SSH user):

```bash
ssh root@200.97.168.12
```

Install Nginx and create the web root:

```bash
apt update
apt install -y nginx
mkdir -p /var/www/nibe-limited
```

Allow inbound TCP ports 22, 80, and (when using a domain and HTTPS) 443 in both the Hostinger VPS firewall and the Ubuntu firewall. If UFW is enabled:

```bash
ufw allow OpenSSH
ufw allow 'Nginx Full'
ufw status
```

## 2. Upload the website

Run these commands from the project root on your Windows computer in PowerShell. `scp` is included with current Windows OpenSSH installations. They upload all site content and the Nginx config; `.git`, `node_modules`, and local environment files are excluded from the archive.

```powershell
tar --exclude=.git --exclude=node_modules --exclude=.env --exclude=.env.* -czf nibe-site.tar.gz .
scp .\nibe-site.tar.gz root@200.97.168.12:/tmp/nibe-site.tar.gz
scp .\deploy\nginx-site.conf root@200.97.168.12:/tmp/nibe-site.conf
```

Then SSH to the VPS and extract/configure/reload:

```bash
tar -xzf /tmp/nibe-site.tar.gz -C /var/www/nibe-limited
chown -R www-data:www-data /var/www/nibe-limited
install -m 0644 /tmp/nibe-site.conf /etc/nginx/sites-available/nibe-limited
ln -sfn /etc/nginx/sites-available/nibe-limited /etc/nginx/sites-enabled/nibe-limited
rm -f /etc/nginx/sites-enabled/default
nginx -t && systemctl reload nginx
```

Visit `http://200.97.168.12` to check the initial deployment. To attach `nibelimited.com`, open the DNS settings at the domain registrar and create these records (remove conflicting records for the same names):

| Type | Name | Value | TTL |
| --- | --- | --- | --- |
| A | `@` | `200.97.168.12` | Default |
| A | `www` | `200.97.168.12` | Default |

If the domain uses custom nameservers, edit DNS at the provider hosting those nameservers. Wait for DNS to resolve to `200.97.168.12`. Then install the updated config from the project (or edit `/etc/nginx/sites-available/nibe-limited` so its `server_name` is `nibelimited.com www.nibelimited.com`) and reload:

```bash
nginx -t && systemctl reload nginx
```

## 3. Enable HTTPS for a domain

After both domain names resolve to the VPS and ports 80 and 443 are allowed, run:

```bash
apt install -y certbot python3-certbot-nginx
certbot --nginx -d nibelimited.com -d www.nibelimited.com
```

Certbot configures the certificate and renewal. Keep ports 80 and 443 open. Both `http://nibelimited.com` and `http://www.nibelimited.com` should be reachable before requesting the certificate.

## Updating the site

Repeat the archive upload and extraction steps, then run `chown -R www-data:www-data /var/www/nibe-limited`. Nginx serves the new files immediately; reload is only needed if the Nginx configuration changes.

## Notes

- This setup does not create a mail handler for the contact page. A static site cannot send form submissions without a backend or third party form service.
- Large video files are served directly by Nginx. Confirm the VPS storage and transfer limits suit the site's media library.
- Paths on Linux are case sensitive, so preserve the repository's file and folder names exactly during upload.
