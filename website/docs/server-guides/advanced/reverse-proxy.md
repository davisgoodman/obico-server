---
title: Configure Obico to work with a reverse proxy
---

:::danger
**Security Warning**: The guide below only cover the basic steps to set up a reverse proxy for self-hosted Obico Server. The setup required to properly secure your reverse proxy is too complicated to be covered here. Please do your own research to gather the necessary info before you proceed.
:::

You can set up a reverse proxy in front of your self-hosted Obico Server.

Two configuration items need to be set differently if you are using a reverse proxy.

## 1. "Domain name" in Django site configuration. {#1-domain-name-in-django-site-configuration}

1. Open Django admin page at `http://tsd_server_ip:3334/admin/`.

2. Login with username `root@example.com`.

3. On Django admin page, click "Sites", and click the only entry "example.com" to bring up the site you need to configure. Set "Domain name" as follows:

Suppose:

* `reverse_proxy_ip`: The public IP address of your reverse proxy. If you use a domain name for the reverse proxy, this should the domain name.
* `reverse_proxy_port`: The port of your reverse proxy.

The "Domain name" needs to be set to `reverse_proxy_ip:reverse_proxy_port`. The `:reverse_proxy_port` part can be omitted if it is standard 80 or 443 port.

## 2. If the reverse proxy is accessed through HTTPS: {#2-if-the-reverse-proxy-is-accessed-through-https}

1. If you haven't already, make a copy of `dotenv.example` and rename it as `.env` in the `obico-server` directory.
2. Open `.env` using your favorite editor.
3. Find `SITE_USES_HTTPS=False` and replace it with `SITE_USES_HTTPS=True`.
4. Restart the server: `docker-compose restart`.

## NGINX {#nginx}

For webcam feed to work, remember to activate Websockets support. Otherwise there will no webfeed when accessing through proxy.

![NginxProxyManagerSettings](/img/server-guides/nginxsettings.png)

Please note that this is not a general guide. Your situation/configuration may be different.

* This configuration does a redirect from port 80 to 443.
* This config is IP agnostic meaning it should work for IPv4 or IPv6.
* This config supports HTTP/2 as well as HSTS TLSv1.3/TLSv1.2, please do note that anything relying on a websocket runs over http1.1.

```
server {
  listen 80;
  listen [::]:80;
  server_name YOUR.PUBLIC.DOMAIN.HERE.com;
  return 301 https://$host$request_uri;
}
server {
  listen 443 ssl http2;
  listen [::]:443 ssl http2;
  ssl_certificate /YOUR/PATH/HERE/fullchain.pem;
  ssl_certificate_key /YOUR/PATH/HERE/privkey.pem;
  ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-SHA384;
  ssl_prefer_server_ciphers on;
  ssl_stapling on;
  ssl_stapling_verify on;
  ssl_protocols TLSv1.3 TLSv1.2;
  ssl_early_data on;
  proxy_set_header Early-Data $ssl_early_data;
  ssl_dhparam /etc/ssl/certs/dhparam.pem;
  ssl_ecdh_curve secp384r1;
  ssl_session_cache shared:SSL:40m;
  ssl_session_timeout 4h;
  add_header Strict-Transport-Security "max-age=63072000;";
  server_name YOUR.PUBLIC.DOMAIN.HERE.com;
  access_log /var/log/tsd.access.log;
  error_log /var/log/tsd.error.log;
  location / {
    proxy_pass http://YOUR BACKEND IP/HOSTNAME:3334/;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header Host $http_host;
    proxy_set_header X-Forwarded-Proto https;
    proxy_redirect off;
    client_max_body_size 10m;
  }
 location /ws/ {
    proxy_pass http://YOUR BACKEND IP/HOSTNAME:3334/ws/;
    proxy_http_version 1.1;
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection "Upgrade";
  }
}
```
## NGINX Proxy Manager - NPM

The setup assumes the following:

- FQDN of the Obico public server: `obico.domain.com`
-	Obico Self-Hosted server IP: `192.168.2.10`
-	NPM IP address: `192.168.2.100`
-	Available public IP address which is either fixed if DHCP is used a DDNS mechanism is in place to update the DNS record of the firewall FQDN.
-	Full access to the DNS server in setting up DNS records in order to get certificate issued automatically via NPM. 
-	In this example Cloudflare is used as the DNS server.
-	Obico Self Hosted server Django site name has been set to : `obico.domain.com`.
-	The .env file has the following environment variable set to:

```
SITE_USES_HTTPS=True
# set it to True if https is set up

SITE_IS_PUBLIC=True
# set it to True if the site configured in django admin is accessible from the internet
```
## 1. Setting up the DNS server:

1. DNS server for domain.com requires the following record:
    `obico	 A  <public IP address>`
   this next record is required to enable the tunnel within the Obico apps or web gui.

    `*.tunnels.obico A <public IP address>` *( this can also be a CNAME record pointing to the obico A record)*

2. In Cloudflare you will need to create an API token to allow updates since Let’s Encrypt in NPM will need to insert some TXT DNS record to verify you have control of domain.com.
   `https://developers.cloudflare.com/fundamentals/api/get-started/create-token/`
   - Select your domain and give it the Edit permission.
   -  Make sure to write down the API Token as it will be needed in NGINX later and you can’t go back, and view again once created. You would have to delete and recreate a new one.

3. Configuring the firewall
   - Port Forwarding:
     - Port 80 and 443 requires to be forwarded to the NPM internal IP address `192.168.2.100`
4. Setting up NPM on docker
   I have used the following to setup NPM:
   
   `https://www.theserverside.com/blog/Coffee-Talk-Java-News-Stories-and-Opinions/How-to-setup-the-Nginx-Proxy-Manager-example#:~:text=Login%20to%20the%20Nginx%20Proxy,Nginx%20Proxy%20Manager%20has%20configured`

   - The docker compose file `docker-compose.yml` contains the following:
```
version: '3.8'
services:
  app:
    image: 'jc21/nginx-proxy-manager:latest'
    restart: unless-stopped
    ports:
      - '80:80'
      - '81:81'
      - '443:443'
    volumes:
      - ./data:/data
      - ./letsencrypt:/etc/letsencrypt
  ```
      

5. Required SSL certificates for using NPM and the Obico self-hosted server.

   - In order to have the Obico self-hosted server available from a public facing IP address and also having the tunnels being created to access the printer WebUI a minimum of 2 certificates will be required:

   - For the main website `obico.domain.com` you will need to create a Let’s Encrypt certificate.
      - You can either create a wildcard certificate `*.domain.com or obico.domain.com` or a certificate that covers both.
   - You will also need to create one more wildcard for `*.tunnels.obico.domain.com`. 
      - You have to create this one by itself as wildcard certificate can only use one subdomain therefore a `*.domain.com` would not cover the `tunnels.obico` subdomain. Failing to do so will prevent the tunnels from           being created.

    - In the NPM webgui under SSL Certificates click “Add SSL Certificates”
      - Make sure to use “DNS Challenge” inserting your previously created API token in the required field.
      - Firewall port forwarding must be enable for port 80 to the NPM host 192.168.2.100 for the DNS Challenge to work.
      
    - Create each certificates, *.tunnels.obico.domain.com and *.obico.domain.com using the following settings and changing the domain names for each:
    - /images/gmail_setup_1-d33c43ec8cf227a0ea16222f21659c10.png

## Traefik {#traefik}

1. [Follow these instructions on how to setup Traefik (First two steps)](https://www.digitalocean.com/community/tutorials/how-to-use-traefik-as-a-reverse-proxy-for-docker-containers-on-debian-9)

1. `cd obico-server`

1. Edit the `docker-compose.override.yml` file with your favorite editor.

1. Add `labels:` and `networks:` to the `web:` section, and also add `networks:` at the end of the file:

    ```yaml
    ...
      web:
        <<: *web-defaults
        hostname: web
        ports:
          - 3334:3334
        labels:
          - traefik.backend=thespaghettidetective
          - traefik.frontend.rule=Host:spaghetti.your.domain
          - traefik.docker.network=web
          - traefik.port=3334
        networks:
          - web
        depends_on:
          - ml_api
        command: sh -c "python manage.py collectstatic --noinput && python manage.py migrate && python manage.py runserver 0.0.0.0:3334"

      ...

      ...

        networks:
          web:
            external: true
      ```

1. Restart the Obico Server with `docker compose restart`

1. You should now be able to browse to `spaghetti.your.domain`
