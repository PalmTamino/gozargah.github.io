```
title: Activating SSL in Marzban
---

# Activating SSL in Marzban

By activating SSL in Marzban, the dashboard and subscription link will be accessible via https.
There are different approaches to activating SSL in Marzban, which are mentioned below in three methods, in order from simple to complex.

::: tip Note
In all the examples below, the `docker-compose.yml` and `.env` files can be found in the `/opt/marzban` path, and `xray_config.json` can be found in the `/var/lib/marzban` path.

If you have installed Marzban manually, you must make the required changes yourself.
:::


## Activating SSL with Caddy

In this method, you do not need to create an SSL certificate; Caddy does all the work for you!


- Change the `docker-compose.yml` file as follows.

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
```
:::

- Create a new file named `Caddyfile` in the `/opt/marzban` path and change `YOUR_DOMAIN` to your desired domain or subdomain.

::: warning Attention
The first letter in the `Caddyfile` filename must be a capital `C`.
:::

::: code-group
```caddy [Caddyfile]
YOUR_DOMAIN {
    reverse_proxy unix//var/lib/marzban/marzban.socket
}
:::


::: warning Attention
If you want the subscription domain or subdomain to be different from the panel, place the above content twice in your Caddyfile and put both domains or subdomains instead of `YOUR_DOMAIN`.
:::

- Set the following variables in the `.env` file.

Change `YOUR_DOMAIN` to your desired domain or subdomain.

```env
UVICORN_UDS = /var/lib/marzban/marzban.socket
XRAY_SUBSCRIPTION_URL_PREFIX = https://YOUR_DOMAIN
```

- Restart Marzban.

```bash
marzban restart
```

Now the Marzban dashboard will be available at your domain or subdomain address via https.


## Activating SSL with Uvicorn

Marzban is run by default with `Uvicorn`. `Uvicorn` also allows you to define SSL certificate files.

- To start, you need to obtain the certificate files for your domain or subdomain. To do this, see the [Creating SSL Certificate](issue-ssl-certificate.md) tutorial.

- After creating the SSL certificate files, set the following variables in the `.env` file.

Change `YOUR_DOMAIN` to your desired domain or subdomain.

```env
UVICORN_PORT = 443
UVICORN_SSL_CERTFILE = "/var/lib/marzban/certs/YOUR_DOMAIN.cer"
UVICORN_SSL_KEYFILE = "/var/lib/marzban/certs/YOUR_DOMAIN.cer.key"
XRAY_SUBSCRIPTION_URL_PREFIX = https://YOUR_DOMAIN
```

Now the Marzban dashboard will be available at your domain or subdomain address via https.


## Activating SSL with HAProxy

`HAProxy` is one of the best tools for this. In this method, we run Marzban with `HAProxy` over https.

- To start, you need to obtain the certificate files for your domain or subdomain. To do this, see the [Creating SSL Certificate](issue-ssl-certificate.md) tutorial.


- Change the `docker-compose.yml` file as follows.

::: code-group
```yml{9-10,12-20} [docker-compose.yml]
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
```
:::

- Create a new file named `haproxy.cfg` in the `/opt/marzban` path and change `YOUR_DOMAIN` to your desired domain or subdomain.

::: code-group
```cfg [haproxy.cfg]
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

Change `YOUR_DOMAIN` to your desired domain or subdomain.

```env
UVICORN_UDS = /var/lib/marzban/marzban.socket
XRAY_SUBSCRIPTION_URL_PREFIX = https://YOUR_DOMAIN
```

- Restart Marzban.

```bash
marzban restart
```

Now the Marzban dashboard will be available at your domain or subdomain address via https.
```
