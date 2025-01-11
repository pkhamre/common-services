# How to set up nginx and certbot on Docker Compose with automatic renewal of SSL certificates

## Step 1 - nginx and initial SSL Certificates

### docker-compose-initial.yml

```yaml
name: common-services

services:

  nginx:
    image: nginx:1.27.3
    volumes:
      - ./config/nginx:/etc/nginx/conf.d:ro
      - ./config/certbot:/etc/nginx/ssl:ro
      - ./public:/usr/share/nginx:ro
    ports:
      - "80:80"
      - "443:443"

  certbot:
    image: certbot/certbot:v3.1.0
    volumes:
      - ./config/certbot:/etc/letsencrypt:rw
      - ./public/certbot:/usr/share/nginx/certbot:rw
```

### config/nginx/www.example.org-NOSSL.conf

```
server {
  listen 80 default_server;
  listen [::]:80 default_server;

  server_name www.example.org;
  server_tokens off;

  location /.well-known/acme-challenge/ {
    root /usr/share/nginx/certbot;
  }

  location / {
    return 301 https://www.example.org$request_uri;
  }
}
```

### Initial Certificate generation

`docker compose run --rm certbot certonly --webroot --webroot-path /usr/share/nginx/certbot -d www.example.org -m address@example.org --agree-tos --no-eff-email`

```shell
root@ubuntu-s-1vcpu-1gb-ams3-01:~/basic-infrastructure# docker compose run --rm certbot certonly --webroot --webroot-path /usr/share/nginx/certbot -d www.example.org -m address@example.org --agree-tos --no-eff-email 
Saving debug log to /var/log/letsencrypt/letsencrypt.log
Requesting a certificate for www.example.org

Successfully received certificate.
Certificate is saved at: /etc/letsencrypt/live/www.example.org/fullchain.pem
Key is saved at:         /etc/letsencrypt/live/www.example.org/privkey.pem
This certificate expires on 2025-04-10.
These files will be updated when the certificate renews.

NEXT STEPS:
- The certificate will need to be renewed before it expires. Certbot can automatically renew the certificate in the background, but you may need to take steps to enable that functionality. See https://certbot.org/renewal-setup for instructions.

- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
If you like Certbot, please consider supporting our work by:
 * Donating to ISRG / Let's Encrypt:   https://letsencrypt.org/donate
 * Donating to EFF:                    https://eff.org/donate-le
- - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - - -
root@ubuntu-s-1vcpu-1gb-ams3-01:~/basic-infrastructure# 
```

## Step 2 - SSL Configuration in nginx

The following URL provides an intermediate, general-purpose server configuration recommended for almost all systems.

https://ssl-config.mozilla.org/#server=nginx&version=1.27.3&config=intermediate&openssl=3.4.0&guideline=5.7

### dhparam

`openssl dhparam -out config/certbot/dhparam.pem 2048`

### nginx config with SSL - config/nginx/www.example.org-SSL.conf

```
server {
  listen 443 ssl;
  listen [::]:443 ssl;

  server_name www.example.org;
  server_tokens off;

  http2 on;

  ssl_certificate /etc/nginx/ssl/live/www.example.org/fullchain.pem;
  ssl_certificate_key /etc/nginx/ssl/live/www.example.org/privkey.pem;

  # HSTS (ngx_http_headers_module is required) (63072000 seconds)
  add_header Strict-Transport-Security "max-age=63072000" always;

  location / {
    root /usr/share/nginx/www.example.org;
  }
}

# intermediate configuration
ssl_protocols TLSv1.2 TLSv1.3;
ssl_ecdh_curve X25519:prime256v1:secp384r1;
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:DHE-RSA-CHACHA20-POLY1305;
ssl_prefer_server_ciphers off;

# see also ssl_session_ticket_key alternative to stateful session cache
ssl_session_timeout 1d;
ssl_session_cache shared:MozSSL:10m;  # about 40000 sessions

# curl https://ssl-config.mozilla.org/ffdhe2048.txt > /path/to/dhparam
ssl_dhparam /etc/nginx/ssl/dhparam;

# OCSP stapling
ssl_stapling on;
ssl_stapling_verify on;

# verify chain of trust of OCSP response using Root CA and Intermediate certs
ssl_trusted_certificate /etc/nginx/ssl/live/www.example.org/chain.pem;

# replace with the IP address of your resolver;
# async 'resolver' is important for proper operation of OCSP stapling
resolver 127.0.0.1;

# If certificates are marked OCSP Must-Staple, consider managing the
# OCSP stapling cache with an external script, e.g. certbot-ocsp-fetcher

# HSTS
server {
  listen 80 default_server;
  listen [::]:80 default_server;

  server_name www.example.org;
  server_tokens off;

  location /.well-known/acme-challenge/ {
    root /usr/share/nginx/certbot;
  }

  location / {
    return 301 https://www.example.org$request_uri;
  }
}
```