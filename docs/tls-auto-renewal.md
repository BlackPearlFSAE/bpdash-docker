# Automatic SSL renewal (Certbot + Let's Encrypt)

> Status: **not enabled** in the current stack. The frontend serves plain HTTP on port 80.
> This doc is a reference for when HTTPS is wired up later.

The `frontend` container will serve HTTPS on `:443` and expect certs at:

```
/etc/letsencrypt/live/yourdomain.com/fullchain.pem
/etc/letsencrypt/live/yourdomain.com/privkey.pem
```

See [../frontend/nginx.conf](../frontend/nginx.conf) — uncomment the HTTPS block and replace `yourdomain.com`.

## 1. Install Certbot on the host (Ubuntu/Debian)

```bash
sudo apt update
sudo apt install -y certbot
```

## 2. Issue the certificate

Use the `standalone` challenge — certbot spins up its own temporary web server on port 80.

> ⚠️ **First deploy:** issue the cert **before** `docker compose up`, while port 80 is free.
> **Subsequent renewals:** stop the frontend to free port 80, then start it again.

```bash
docker compose stop frontend                          # only if already running
sudo certbot certonly --standalone -d yourdomain.com
docker compose start frontend                         # only if you stopped it
```

Certs land in `/etc/letsencrypt/live/yourdomain.com/` on the host.

## 3. Expose the certs to the frontend container

Uncomment the `443:443` port and the `/etc/letsencrypt` bind mount in [../compose.yml](../compose.yml), then:

```bash
docker compose up -d frontend
```

## 4. Auto-renewal

Certbot installs a systemd timer (`certbot.timer`) that runs twice daily. Because nginx is containerized, add a deploy hook so the container reloads after renewal:

```bash
sudo mkdir -p /etc/letsencrypt/renewal-hooks/deploy
echo -e '#!/bin/bash\ndocker compose -f <path to compose.yml> exec frontend nginx -s reload' \
  | sudo tee /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
sudo chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
```

Test renewal end-to-end:

```bash
sudo certbot renew --dry-run
```
