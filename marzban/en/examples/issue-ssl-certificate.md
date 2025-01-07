Here is the translation of your file while maintaining the original .md formatting:

---
title: Enabling SSL in Marzban
---

# Enabling SSL in Marzban

By enabling SSL in Marzban, the dashboard and subscription link will be accessible via HTTPS. There are different approaches to enabling SSL in Marzban, and we will explain three methods below, ordered from simplest to most complex.

::: tip Note
In all examples below, the `docker-compose.yml` and `.env` files can be found in the `/opt/marzban‍‍‍` directory, and the `xray_config.json` file is in the `/var/lib/marzban` directory.

If you installed Marzban manually, you must make the necessary changes yourself.
:::

## Enabling SSL with Caddy

In this method, you don't need to create an SSL certificate; Caddy handles everything for you!

- Modify the `docker-compose.yml` file as follows:

::: code-group
```yml{9-10,12-22,24-25} [docker-compose.yml]
services:
  marzban:
    image: gozargah/marzban:latest
    restart: always
    env_file: .env
    network_mode: host
    volumes:
      - /var/lib/marzban:/var/lib/marzban
    depends_on:
      - caddy

  caddy:
    image: caddy
    restart: always
    ports:
      - 80:80
      - 443:443
    volumes:
      - /var/lib/marzban:/var/lib/marzban
      - ./Caddyfile:/etc/caddy/Caddyfile
      - caddy_volume:/data
      - caddy_volume:/config

volumes:
  caddy_volume:

:::
	•	Create a new file named Caddyfile in the /opt/marzban directory and replace YOUR_DOMAIN with your desired domain or subdomain.

::: warning Note
The first letter in the filename Caddyfile must be uppercase.
:::

::: code-group

YOUR_DOMAIN {
	reverse_proxy unix//var/lib/marzban/marzban.socket
}
:::

::: warning Note
If the domain or subdomain for the subscription differs from the panel, duplicate the above content in your Caddyfile and replace `YOUR_DOMAIN` with both domains or subdomains.
:::

- Set the following variables in the `.env` file.

Replace `YOUR_DOMAIN` with your desired domain or subdomain.

```env
UVICORN_UDS = /var/lib/marzban/marzban.socket
XRAY_SUBSCRIPTION_URL_PREFIX = https://YOUR_DOMAIN

	•	Restart Marzban.

marzban restart

Now, the Marzban dashboard will be accessible via HTTPS at your domain or subdomain.

Enabling SSL with Uvicorn

Marzban uses Uvicorn by default. Additionally, Uvicorn allows you to define SSL certificate files.
	•	First, obtain certificate files for your domain or subdomain. For this, see the SSL Certificate Creation guide.
	•	After creating the SSL certificate files, set the following variables in the .env file.

Replace YOUR_DOMAIN with your desired domain or subdomain.

UVICORN_PORT = 443
UVICORN_SSL_CERTFILE = "/var/lib/marzban/certs/YOUR_DOMAIN.cer"
UVICORN_SSL_KEYFILE = "/var/lib/marzban/certs/YOUR_DOMAIN.cer.key"
XRAY_SUBSCRIPTION_URL_PREFIX = https://YOUR_DOMAIN

Now, the Marzban dashboard will be accessible via HTTPS at your domain or subdomain.

Enabling SSL with HAProxy

HAProxy is one of the best tools for this task. In this method, Marzban is run with HTTPS using HAProxy.
	•	First, obtain certificate files for your domain or subdomain. For this, see the SSL Certificate Creation guide.
	•	Modify the docker-compose.yml file as follows:

::: code-group

services:
  marzban:
      image: gozargah/marzban:latest
      restart: always
      env_file: .env
      network_mode: host
      volumes:
        - /var/lib/marzban:/var/lib/marzban
      depends_on:
        - haproxy
    
  haproxy:
    image: haproxy:latest
    restart: always
    volumes:
      - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
      - /var/lib/marzban:/var/lib/marzban
    ports:
      - 80:80
      - 443:443

:::
	•	Create a new file named haproxy.cfg in the /opt/marzban directory and replace YOUR_DOMAIN with your desired domain or subdomain.

::: code-group

defaults
  mode tcp
  timeout client 30s
  timeout connect 4s
  timeout server 30s

global
  maxconn 10000000

frontend http_frontend
  bind *:80
  mode http
  redirect scheme https code 301 if !{ ssl_fc }

frontend https_frontend
  bind *:443 ssl crt /var/lib/marzban/certs/YOUR_DOMAIN.cer
  default_backend marzban_backend

backend marzban_backend
  server marzban /var/lib/marzban/marzban.socket
:::

- Set the following variables in the `.env` file.

Replace `YOUR_DOMAIN` with your desired domain or subdomain.

```env
UVICORN_UDS = /var/lib/marzban/marzban.socket
XRAY_SUBSCRIPTION_URL_PREFIX = https://YOUR_DOMAIN

	•	Restart Marzban.

marzban restart

Now, the Marzban dashboard will be accessible via HTTPS at your domain or subdomain.

Let me know if there are any further adjustments or clarifications needed!
