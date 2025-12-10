# cheshire-cat-nginx-proxy
Cheshire Cat AI with NGINX Reverse Proxy and SSL
---

This guide explains how to configure [Cheshire Cat AI](https://cheshirecat.ai/) to run behind an NGINX reverse proxy with HTTPS using Docker Compose. This setup ensures both the API and UI are served securely under HTTPS.

## Prerequisites
To follow this guide, you’ll need:
- A domain: Replace `mydomain.org` with your actual domain name.
- A virtual machine (VM): A server or VM exposed to the internet or other configuration of your preference.
- **Docker Compose**: Ensure Docker Compose is installed on your VM.

## Step 1: Set Up the Project Directory
Create a directory to hold our project files:
```bash
$ mkdir cheshire-cat-nginx-proxy
$ cd cheshire-cat-nginx-proxy
```
Create the following files within this directory:
- `compose.yaml`
- `.env`
- `nginx/default.conf`

## Step 2: Edit the `compose.yaml` File
This file defines the services used in our setup.

**Note**: In this configuration, no ports from the Cat container are exposed to the host system. Communication between the Cat and NGINX containers occurs exclusively within the default Docker bridge network, ensuring an isolated internal connection.

```yaml
services:

  cheshire-cat-core:
    restart: always
    image: ghcr.io/cheshire-cat-ai/core:latest
    container_name: cheshire_cat_core
    volumes:
      - ./static:/app/cat/static
      - ./plugins:/app/cat/plugins
      - ./data:/app/cat/data
    env_file:
      # environment variables file named .env has to be in the same folder of compose.yaml
      - .env

  nginx:
    restart: always
    image: nginx:latest
    ports:
      - 80:80
      - 443:443
    volumes:
      - ./nginx/default.conf:/etc/nginx/conf.d/default.conf
      - ./certbot/www:/var/www/certbot/:ro
      - ./certbot/conf/:/etc/nginx/ssl/:ro

  certbot:
    image: certbot/certbot:latest
    volumes:
      - ./certbot/www/:/var/www/certbot/:rw
      - ./certbot/conf/:/etc/letsencrypt/:rw
```

## Step 3: Configure Environment Variables in `.env`
These environment variables tell the Cat how to behave when behind a proxy
```
CCAT_CORE_HOST=mydomain.org
CCAT_CORE_USE_SECURE_PROTOCOLS=true
CCAT_HTTPS_PROXY_MODE=true
# next env vars are suggested to secure the installation as reported https://cheshire-cat-ai.github.io/docs/production/auth/authentication/
CCAT_API_KEY=a-very-long-and-alphanumeric-secret
CCAT_API_KEY_WS=another-very-long-and-alphanumeric-secret
CCAT_JWT_SECRET=yet-another-very-long-and-alphanumeric-secret
```

## Step 4: Set Up NGINX Configuration in `./nginx/default.conf`
Edit the `./nginx/default.conf` file to include this configuration (**remember**: replace `mydomain.org` with the real domain!!!)

```
server {
    listen       80;
    listen  [::]:80;
    server_name  mydomain.org;
    server_tokens off;

    location / {
        return 301 https://mydomain.org$request_uri;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location /ws {
        return 301 https://mydomain.org$request_uri;
    }
}
```

## Step 5: Request an SSL Certificate
Start the services (excluding certbot):

```bash
docker compose up -d nginx cheshire-cat-core
```
Request an SSL certificate using Certbot:
```bash
docker compose run --rm  certbot certonly --webroot --webroot-path /var/www/certbot/ -d mydomain.org
```

## Step 6: update the NGINX configuration for https
Edit the `./nginx/default.conf` file to include the SSL configuration (**remember**: replace `mydomain.org` with the real domain!!!)

```
server {
    listen       80;
    listen  [::]:80;
    server_name  mydomain.org;
    server_tokens off;

    location / {
        return 301 https://mydomain.org$request_uri;
    }

    location /.well-known/acme-challenge/ {
        root /var/www/certbot;
    }

    location /ws {
        return 301 https://mydomain.org$request_uri;
    }
}

server {
    listen 443 default_server ssl;
    listen [::]:443 ssl;
    http2 on;

    server_name mydomain.org;

    ssl_certificate /etc/nginx/ssl/live/mydomain.org/fullchain.pem;
    ssl_certificate_key /etc/nginx/ssl/live/mydomain.org/privkey.pem;

    location / {
        proxy_pass http://cheshire-cat-core/;
        proxy_http_version 1.1;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /ws {
        proxy_pass http://cheshire-cat-core/ws;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "Upgrade";
        proxy_set_header Host $host;
    }
}
```

## Step 7: Restart NGINX with SSL Configuration
Restart the NGINX service to apply the SSL configuration:
```bash
docker compose restart nginx
```

## To Renew the SSL certificate
The Let'sEncrypt certificates expired in 3 month. Before expiration run
```bash
$ docker compose run --rm  certbot renew
```
