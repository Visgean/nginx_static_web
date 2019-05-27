# Nginx for static web hosting

Sources: 
- https://gist.github.com/nrollr/9a39bb636a820fb97eec2ed85e473d38
- https://www.digitalocean.com/community/questions/how-can-i-configure-nginx-to-deliver-static-html-pages
- https://medium.com/@jgefroh/a-guide-to-using-nginx-for-static-websites-d96a9d034940


# What does this give you:

- secure https and http2 
- caching on client side
- *about.html -> /about/* rewrite so you have those cool urls




# Setting up server

I recommend you create a new Ubuntu Droplet using [this referal link](https://m.do.co/c/382cb33740a4). I think the 25GB Droplet for $5 is more than enough.

## Mail

If you are using Google domains you can set up mail forwarding by following this guide: https://support.google.com/domains/answer/3251241?hl=en. note this works even if you are not using their name servers - I use Digital Ocean name servers and it works just fine.

## Firewall:

```
ufw allow ssh
ufw allow https
ufw allow http 
ufw enable
```

##  New user:

Create non-root user that will be used for uploading the site

```
adduser <username>
usermod -aG sudo <username>
```

Create folder for the static site:

```
cd /var/www/
mkdir mysite
chown -R <username> mysite
```

On your local machine use this command to copy ssh key on the remote server
```
ssh-copy-id <username>@<remote>
```


# nginx

Install it:

```
sudo apt-get update
sudo apt-get install nginx
```


# nginx template:

You need to replace all ``example.com`` with your own value and ``/var/www/example`` with the actual directory

```nginx
# UPDATED 17 February 2019
# see https://gist.github.com/nrollr/9a39bb636a820fb97eec2ed85e473d38
# and https://github.com/Visgean/nginx_static_web

# Redirect all HTTP traffic to HTTPS
server {
    listen 80;
    listen [::]:80;
    server_name www.example.com example.com;
    return 301 https://$host$request_uri;
}

# SSL configuration
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name www.example.com example.com;
    ssl_certificate      /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key  /etc/letsencrypt/live/example.com/privkey.pem;

    # Improve HTTPS performance with session resumption
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 10m;

    # Enable server-side protection against BEAST attacks
    ssl_protocols TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES256-SHA384:ECDHE-RSA-AES256-SHA384";

    # RFC-7919 recommended: https://wiki.mozilla.org/Security/Server_Side_TLS#ffdhe4096
    #    ssl_dhparam /etc/ssl/ffdhe4096.pem;
    #    ssl_ecdh_curve secp521r1:secp384r1;

    # Aditional Security Headers
    # ref: https://developer.mozilla.org/en-US/docs/Security/HTTP_Strict_Transport_Security
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains";

    # ref: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Frame-Options
    add_header X-Frame-Options DENY always;

    # ref: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-Content-Type-Options
    add_header X-Content-Type-Options nosniff always;

    # ref: https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/X-XSS-Protection
    add_header X-Xss-Protection "1; mode=block" always;

    # Enable OCSP stapling
    # ref. http://blog.mozilla.org/security/2013/07/29/ocsp-stapling-in-firefox
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    resolver 1.1.1.1 1.0.0.1 [2606:4700:4700::1111] [2606:4700:4700::1001] valid=300s; # Cloudflare
    resolver_timeout 5s;

    gzip on;
    gzip_types application/javascript image/* text/css;
    gunzip on;

    location ~* \.(jpg|jpeg|png|gif|ico)$ {
        expires 30d;
    }
    location ~* \.(css|js)$ {
        expires 7d;
    }

    # Required for LE certificate enrollment using certbot
    location '/.well-known/acme-challenge' {
        default_type "text/plain";
        root /var/www/html;
    }

    index index.html;
    root /var/www/example;

    location / {
        try_files $uri $uri/ @htmlext;
    }

    location ~ \.html$ {
        try_files $uri =404;
    }

    location @htmlext {
        rewrite ^(.*)$ $1.html last;
    }
}
```

copy the template to ``/etc/nginx/sites-available/<mysite>`` and create symlink to it in ``sites-enabled``:

```
ln -s /etc/nginx/sites-available/<mysite> /etc/nginx/sites-enabled/<mysite>
```

# Lets encrypt

```
sudo apt-get install software-properties-common
sudo add-apt-repository ppa:certbot/certbot
sudo apt-get update
sudo apt-get install python-certbot-nginx
```

Now run this command, *it will most likely fail to identify your domains* so you have to enter them manually - you will have to enter something like ``example.com www.example.com``
```
sudo certbot --nginx certonly
```

# Use rsync to publish your website 

Rsync is faster than SFTP because it only transmits the changed items

```
rsync -azP <build_dir on your laptop> <digital ocean ssh serve details>:/var/www/<mysite>
```

# Restart nginx:

```
systemctl restart nginx
```
