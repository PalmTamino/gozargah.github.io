

### Enabling SSL in Marzban

By enabling SSL in Marzban, the dashboard and subscription links will be accessible via HTTPS. Below are three methods for enabling SSL in Marzban, ordered from simplest to most complex.

::: tip Note
In all the examples below:
- The `docker-compose.yml` and `.env` files can be found in the `/opt/marzban` directory.
- The `xray_config.json` file can be found in the `/var/lib/marzban` directory.

If you installed Marzban manually, you must make the required changes yourself.
:::

---

### Enabling SSL with Caddy

In this method, you don’t need to create an SSL certificate—Caddy will handle everything for you.

1. Modify the `docker-compose.yml` file as follows:

```yml
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

2. Create a new file named `Caddyfile` in the `/opt/marzban` directory. Replace `YOUR_DOMAIN` with your desired domain or subdomain.

::: warning Note
The file name must begin with a capital `C`.
:::

```caddy
YOUR_DOMAIN {
    reverse_proxy unix//var/lib/marzban/marzban.socket
}
```

::: warning Note
If the subscription domain/subdomain is different from the panel domain, add the configuration for both domains in the `Caddyfile`.
:::

3. Update the following variables in the `.env` file. Replace `YOUR_DOMAIN` with your domain or subdomain:

```env
UVICORN_UDS=/var/lib/marzban/marzban.socket
XRAY_SUBSCRIPTION_URL_PREFIX=https://YOUR_DOMAIN
```

4. Restart Marzban:

```bash
marzban restart
```

Your Marzban dashboard will now be accessible via HTTPS at your domain or subdomain.

---

### Enabling SSL with Uvicorn

Marzban runs using `Uvicorn` by default, which also supports SSL certificate configuration.

1. Obtain the SSL certificate files for your domain or subdomain. See the guide on [Issuing an SSL Certificate](issue-ssl-certificate.md).

2. After obtaining the SSL certificate files, update the following variables in the `.env` file. Replace `YOUR_DOMAIN` with your domain or subdomain:

```env
UVICORN_PORT=443
UVICORN_SSL_CERTFILE="/var/lib/marzban/certs/YOUR_DOMAIN.cer"
UVICORN_SSL_KEYFILE="/var/lib/marzban/certs/YOUR_DOMAIN.cer.key"
XRAY_SUBSCRIPTION_URL_PREFIX=https://YOUR_DOMAIN
```

3. Restart Marzban:

```bash
marzban restart
```

Your Marzban dashboard will now be accessible via HTTPS at your domain or subdomain.

---

### Enabling SSL with HAProxy

`HAProxy` is a robust tool for enabling HTTPS. In this method, Marzban is configured to use `HAProxy` for HTTPS.

1. Obtain the SSL certificate files for your domain or subdomain. See the guide on [Issuing an SSL Certificate](issue-ssl-certificate.md).

2. Modify the `docker-compose.yml` file as follows:

```yml
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

3. Create a new file named `haproxy.cfg` in the `/opt/marzban` directory. Replace `YOUR_DOMAIN` with your desired domain or subdomain:

```cfg
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
```

4. Update the following variables in the `.env` file. Replace `YOUR_DOMAIN` with your domain or subdomain:

```env
UVICORN_UDS=/var/lib/marzban/marzban.socket
XRAY_SUBSCRIPTION_URL_PREFIX=https://YOUR_DOMAIN
```

5. Restart Marzban:

```bash
marzban restart
```

Your Marzban dashboard will now be accessible via HTTPS at your domain or subdomain.
