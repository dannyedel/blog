---
tags: automation
title: Installing nginx with a letsencrypt certificate on a fresh Debian jessie VM
---

Starting point:

* Debian jessie installation
* The `jessie-backports` repository is activated in the `sources.list`:

  ```
 deb http://ftp2.de.debian.org/debian jessie-backports main
  ```
* A public hostname is configured in DNS

## Install nginx and letsencrypt software

This uses `nginx` from stable, but `letsencrypt` client is only in backports.

```bash
apt install -t jessie-backports letsencrypt
apt install nginx
```

## Append nginx' SSL config a bit

See [this post about the cipher string][cipher] for the reasoning
behind this exact cipher list.

The debian default `/etc/nginx/nginx.conf` already contains settings
that disable SSLv2/v3 and activate the "Prefer server cipher list"
option.  If you specify them twice, $stuff will break.

```bash
openssl dhparam -out /etc/nginx/dh2048.pem 2048
cat > /etc/nginx/conf.d/ssl-parameters.conf <<- 'EOF'
    ssl_dhparam /etc/nginx/dh2048.pem;
    ssl_ciphers "HIGH+EECDH:HIGH+EDH:@STRENGTH";
    ssl_stapling on;
    ssl_session_cache builtin:1000 shared:SSL:10m;
    add_header Strict-Transport-Security 'max-age=31536000; includeSubDomains;' ;
EOF
```

[cipher]: /2015/11/30/ssl-cipher-suites/

## Get a lets encrypt certificate

**State 2016-04-17: This currently needs IPv4,
IPv6 support is [supposed to come berfore April 22][upcoming]**

This assumes that `hostname -f` will output the FQDN and that the
the certificate is used for only this hostname.  Append as many
`-d other.host.name.fqdn` as you want.

```bash
letsencrypt certonly --webroot -w /var/www/html \
    -d $(hostname -f)
```

[upcoming]: https://letsencrypt.org/upcoming-features/

## Install certificate into nginx

The first `-d` parameter to `letsencrypt certonly` will determine
the filename.  Additional `-d`s will be included in the certificate
as "Subject Alternative Names", allowing for multihosting without needing
SNI.

```bash
cat > /etc/nginx/conf.d/ssl-certificates.conf <<- EOF
    ssl_certificate /etc/letsencrypt/live/$(hostname -f)/fullchain.pem ;
    ssl_certificate_key /etc/letsencrypt/live/$(hostname -f)/privkey.pem ;
EOF
```

## Activate SSL on the default host

In the default config, listening on SSL is commented out
on the default host.

```bash
sed -e 's/# listen 443/listen 443/g' -i /etc/nginx/sites-enabled/default
sed -e 's/# listen \[::\]:443/listen [::]:443/g' -i /etc/nginx/sites-enabled/default
```

## Check config and restart

Run `nginx -t` to check that the config file is all right,
then issue `systemctl restart nginx` and the host should have SSL activated.

