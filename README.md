
# Howto: Automatic SSL certificates for Nginx

Sample code and instructions on how to automatically obtain and update Let's Encrypt SSL certificates for nginx, with no manual steps or maintenance.

## Introduction

Nowadays SSL certificates and HTTPS are mandatory for websites of any scale, as most browsers are starting to mark as "unsecure" plain HTTP connections. Certificates must be issued by a trusted Certificate Authority, a service that is offered free-of-charge by Let's Encrypt, that also provides a command line utility to request or revoke certificates. These actions that are authorized only after the domain is validated. This entire procedure is automatized by some specialized http servers like [Caddy](https://github.com/mholt/caddy/), while it must be performed manually when dealing with any other server, like Nginx; it roughly consists in:
* calling the command `certbot`;
* following an interactive dialogue, that requires an email, the domains and a validation method (spinning up a dedicated server or providing a web-accessible folder);
* waiting the authorization, then edit the nginx file to include the newly issued certificates;
* restarting nginx.

Since i got tired of repeating this scheme every time i have to setup a new website, i wrote a little script that, if run before Nginx, automatizes the procedure by taking the needed data from the Nginx configuration files.

## Setup

Server blocks in Nginx configuration must be setup normally and pointing to the standard location of a Let's Encrypt certificate:
```
server {
  server_name mydomain.com www.myotherdomain.com;
  ssl_certificate /etc/letsencrypt/live/mydomain.com/fullchain.pem;
  ssl_certificate_key /etc/letsencrypt/live/mydomain.com/privkey.pem;
  ...
}
```

Ensure you have the following depencies installed:
* certbot
* GNU grep

The script must be run ***before starting Nginx***. I use it as entrypoint of a Docker image (and then i start Nginx with `nginx -g 'daemon off;'`), but it can also be inserted in a systemd unit as a requirement to Nginx.

```bash
#!/bin/sh

# for each server block
cat /etc/nginx/nginx.conf /etc/nginx/conf.d/* \
    | sed 's/#.*$//' | awk '
    c && /\{/ { c++ }
    /server *\{/ { c=1; print "SERVER_BEGIN" }
    c { print }
    c == 1 && /\}/ { print "SERVER_END" }
    c && /\}/ { c-- }
    ' | tr '\n' ' ' | grep -o -P 'SERVER_BEGIN.+?SERVER_END' | while read BLOCK; do

    # check if block uses certbot and if the certificate does not exist
    CERT_PATH=$(echo $BLOCK | grep -o -P 'ssl_certificate \/etc\/letsencrypt\/live\/.+\/fullchain.pem;' \
        | sed 's/^ssl_certificate //' | sed 's/;$//')
    [ -n "$CERT_PATH" ] && ! [ -f "$CERT_PATH" ] || continue

    DOMAINS=$(echo $BLOCK | grep -o -P 'server_name .+?;' \
        | sed 's/^server_name //' | sed 's/;$//' | tr ' ' ',')

    # request a new certificate
    certbot certonly \
        --non-interactive \
        --agree-tos \
        -m web@example$(((RANDOM % 900) + 100)).com \
        --standalone \
        --domains $DOMAINS \
        || exit 1
done
```

What does it do:
* Nginx configurations files from `/etc/nginx/conf.d` and `/etc/nginx/nginx.conf` are scanned;
* Comments are removed (`sed 's/#.*$//'`);
* Server blocks (`server { ... }`) are extracted;
* If a server uses a certbot certificate, and this certificate does not exists, certbot is called to obtain a certificate;
* When the procedure is finished, nginx is started.

## Renewal

Let's Encrypt certificates must be renewed every 3-4 months. Nginx must not be running during the procedure. Both actions can be scheduled through cron, by adding this entry to cron tabs:
```
3 44 */5 * *  kill -TERM $(cat /run/nginx.pid); certbot -q renew; (nginx -g 'daemon off;' &)
```

