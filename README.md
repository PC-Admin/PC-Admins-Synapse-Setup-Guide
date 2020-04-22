
# PC-Admin's Synapse Setup Guide v2


This guide covers complete Synapse setup for Debian 10 with Postgresql. It includes the often missing sections on how to configure postgresql and coturn with Synapse. As well as additional steps to configure a jitsi instance for conferencing. You can use this guide to make your own encrypted chat server.

You will need at least a 1GB VPS although I recommend 2GB for a small server. You will also need a desired domain name. My guide will use ‘example.org’ with Riot-Web hosted through NGINX on the same server. You may wish to have your matrix service hosted at another prefix like ‘matrix.example.org’, although you probably shouldn't make that your server name. The server name is the public facing name for your server and it can't be changed later. You wouldn't have an email address like jessica@email.gmail.com so don't use a Matrix server name like matrix.example.org.

Join the discussion at: #synapsesetupguide:matrix.org if you get stuck or have an edit in mind.
***
## Licensing

This work is licensed under Creative Commons Attribution Share Alike 4.0, for more information on this license see here: https://creativecommons.org/licenses/by-sa/4.0/
***
## Server Setup

Configure a Debian 10 server with auto-updates, security and SSH access. Ports 80, 443, 8448 and 3478 will need to be open for the web service, synapse federation and the coturn service.
***
## DNS Records

Set up a simple A record. With ‘example.org’ pointed to your servers IP. 
Additionally you might setup a DNS SRV record, though it's only necessary, when you changed your federation port to listen on another port the the default port 8448.

Example DNS SRV record:
```
_matrix._tcp        3600 IN SRV     10 0 8448 example.org
```

***
## Installing Synapse

Follow [the official Debian install instructions](https://github.com/matrix-org/synapse/blob/master/INSTALL.md#debianubuntu) using the matrix.org packages.

You will be asked to set the name of your server after `apt install`, enter your desired URL here. (eg: example.org)

Finally check that the synapse server is shutdown

`$ sudo systemctl stop matrix-synapse`

***
## Installing Postgresql
The default synapse install generates a config that uses sqlite. It has the advantage of being easy to setup as there's no db server setup to take care about. But from my experience the performance penalty is quite big and if you want to do something more then testing or running a small non federated server, switching to postgres should be a mandatory step.

So let's install postgresql:

`$ sudo apt install postgresql postgresql-client`

Create Role and Database

`$ sudo -i -u postgres`

`$ createuser synapse -P --interactive`
```
postgres@VM:~$ createuser synapse -P --interactive
Enter password for new role: 
Enter it again: 
Shall the new role be a superuser? (y/n) n
Shall the new role be allowed to create databases? (y/n) n
Shall the new role be allowed to create more new roles? (y/n) n
```
Remember the db user password, you'll need it later

Now we're back at $postgres. Let's create a database for Synapse with correct settings and set the owner to be the user we just created:

Type: `psql`

..And create the database as follows:

`postgres=# CREATE DATABASE synapse WITH ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0 OWNER synapse;`

Exit from psql by typing `'\q'` 

All done. Let's exit from postgres account by typing 'exit' so land back at our own user.

***
## Adapt Synapse config to use Postgresql
Now as we have created the db and a user to be able to connect, we need to change the synapse config to use it:
`$ sudo nano /etc/matrix-synapse/homeserver.yaml`

Before the change it should look like this:
```
# Database configuration
database:
  # The database engine name
  name: "sqlite3"
  # Arguments to pass to the engine
  args:
    # Path to the database
    database: "/var/lib/matrix-synapse/homeserver.db"
```

Modify it to look like this:
```
database:
    name: psycopg2
    args:
        user: synapse
        password: "your-db-user-password"
        database: synapse
        host: localhost
        cp_min: 5
        cp_max: 10
```

***
## Certbot Setup

`$ sudo apt install certbot`

Test if server IP can be pinged first, if it can then run:

`$ sudo certbot certonly --rsa-key-size 4096`

Choose ‘spin up a temporary webserver’

enter a recovery email

enter ‘example.org’ as the domain

```
IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at:
   /etc/letsencrypt/live/example.org/fullchain.pem
   Your key file has been saved at:
   /etc/letsencrypt/live/example.org/privkey.pem
   Your cert will expire on 2019-11-01.
```

***
## Setup SSL Auto-renewal

for monthly renewal, set a crontab:

`$ sudo crontab -e`

Insert Line:

`@monthly certbot renew --rsa-key-size 4096 --quiet --post-hook "systemctl reload nginx"`

***
## Configure NGINX with A+ SSL

Generate dhparam key and move it to your letsencrypt folder:
```
$ openssl dhparam -out dhparam4096.pem 4096
$ sudo mv ./dhparam4096.pem /etc/letsencrypt/live/example.org
$ sudo chown root:root /etc/letsencrypt/live/example.org/dhparam4096.pem
```
Install NGINX and configure:
```
$ sudo apt install nginx -y
$ sudo nano /etc/nginx/conf.d/matrix.conf
```
Add:

```nginx
server {
       listen         80;
       server_name    example.org;
       return         301 https://$server_name$request_uri;
}

server {
    listen 443 ssl http2;
    listen 8448 ssl http2;  # for federation (skip if pointing SRV to port 443)
    gzip off;
    server_name example.org;

    ssl_certificate     /etc/letsencrypt/live/example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.org/privkey.pem;
    ssl_session_cache   shared:NGX_SSL_CACHE:10m;
    ssl_session_timeout 12h;
    ssl_protocols       TLSv1.3 TLSv1.2;
    ssl_ciphers		"TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256";
    ssl_dhparam         /etc/letsencrypt/live/example.org/dhparam4096.pem;
    ssl_ecdh_curve      X25519:secp521r1:secp384r1:prime256v1;

    add_header Strict-Transport-Security "max-age=63072000; includeSubdomains" always;
    add_header X-Content-Type-Options "nosniff" always;

    location /_matrix {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

^ Make sure to replace the server name here with yours!

Restart service and renew SSL:

`$ sudo certbot renew --rsa-key-size 4096 --quiet --post-hook "systemctl reload nginx"`

If you get a 'Cert not yet due for renewal' error wait a few hours and try again.


Now synapse should be ready and we can see if it starts without errors:

`$ sudo systemctl start matrix-synapse`

`$ sudo systemctl status matrix-synapse`

***
## Fine Tune Synapse
There're two files that manage the behaviour of synapse:
 
- Server config file in /etc/matrix-synapse/homeserver.yaml
- Env file in /etc/default/matrix-synapse

The first is used to do the configuration of synapse, the second is used to setup the environment synapse is running in.

### Registration and guest access

- Remove TLS listener

  We've configured nginx to reverse proxy our federation traffic, so we can remove/comment out the federation listener.

  If you can find the following...

    File: /etc/matrix-synapse/homeserver.yaml: 
```yaml
#  - port: 8448
#    type: http
#    tls: true
#    bind_addresses: ['::', '0.0.0.0']
#    x_forwarded: false
#    resources:
#      - names: [client, federation]
#    compress: true
```

  remove it.

- Remove TLS certs

  We won't have a TLS listener, so we can remove this too.

    File: /etc/matrix-synapse/homeserver.yaml: 
```yaml
#    tls_certificate_path: "/etc/matrix-synapse/fullchain.pem"
#    tls_private_key_path: "/etc/matrix-synapse/privkey.pem"
```

- Federation Blacklist (optional):

    File:  /etc/matrix-synapse/homeserver.yaml: 
```
    federation_ip_range_blacklist:
    #  - '127.0.0.0/8'
    #  - '10.0.0.0/8'
    #  - '172.16.0.0/12'
    #  - '192.168.0.0/16'
    #  - '100.64.0.0/10'
    #  - '169.254.0.0/16'
    #  - '::1/128'
    #  - 'fe80::/64'
    #  - 'fc00::/7'
```

- Allow public room list to federate (optional)

    File:  /etc/matrix-synapse/homeserver.yaml: `allow_public_rooms_over_federation: true`

- Registration (optional)

    File:  /etc/matrix-synapse/homeserver.yaml: `enable_registration: true`

- Admin email (optional)

    File: /etc/matrix-synapse/homeserver.yaml: `admin_contact: 'mailto:admin@example.org’`


**There are other settings here you may want to adjust. I would do so one at a time, testing each change as you go.**

### Cache factor

For a small server (<=2GB), an adoption of the cache factor might improve performance. Some time ago the advice was to lower the cache factor to use less RAM. Experience has shown that the effect is quite the opposite, see [Issue](https://github.com/matrix-org/synapse/pull/4276).

So the new advice is to **raise the cache factor to use less RAM**, with a value of 2 being a good starting point:

- Cache factor

    File: /etc/default/matrix-synapse: SYNAPSE_CACHE_FACTOR=2.0
 
***

<span style="color:red">**Don't forget to restart synapse and examine the RAM usage after each change:**</span>

`$ sudo systemctl restart matrix-synapse`

***
## Load Riot-Web client into NGINX

NGINX content location: /usr/share/nginx/html/index.html

https://github.com/vector-im/riot-web/releases/latest

Download latest riot-web and move contents into nginx folder:
```
$ wget https://packages.riot.im/riot-release-key.gpg
$ gpg --import riot-release-key.gpg
$ wget https://github.com/vector-im/riot-web/releases/download/v1.3.0/riot-v1.3.0.tar.gz
$ wget https://github.com/vector-im/riot-web/releases/download/v1.3.0/riot-v1.3.0.tar.gz.asc
$ gpg --verify riot-v1.3.0.tar.gz.asc riot-v1.3.0.tar.gz
    gpg: Signature made Thu 18 Jul 2019 10:57:59 AM EDT
    gpg:                using RSA key 5EA7E0F70461A3BCBEBE4D5EF6151806032026F9
    gpg:                issuer "releases@riot.im"
    gpg: Good signature from "Riot Releases <releases@riot.im>" [unknown]
$ tar -zxvf ./riot-v1.3.0.tar.gz
$ sudo rm -r /usr/share/nginx/html/*
$ sudo mv ./riot-v1.3.0/* /usr/share/nginx/html/
$ rm -r ./riot-v*
```
Create and edit config.json in nginx directory:

Feel free to customize config.json to suit your needs. All of the lines in config.json are optional.
```
$ sudo cp /usr/share/nginx/html/config.sample.json /usr/share/nginx/html/config.json

$ sudo nano /usr/share/nginx/html/config.json

{
    "default_server_config": {
        "m.homeserver": {
            "base_url": "https://example.org",
            "server_name": "example.org"
        },
        "m.identity_server": {
            "base_url": "https://vector.im"
        }
    },
    "disable_identity_server": false,
    "disable_custom_urls": false,
    "disable_guests": false,
    "disable_login_language_selector": false,
    "disable_3pid_login": false,
    "brand": "Riot",
    "integrations_ui_url": "https://scalar.vector.im/",
    "integrations_rest_url": "https://scalar.vector.im/api",
    "integrations_jitsi_widget_url": "https://scalar.vector.im/api/widgets/jitsi.html",
    "bug_report_endpoint_url": "https://riot.im/bugreports/submit",
    "defaultCountryCode": "AU",
    "showLabsSettings": false,
    "features": {
        "feature_groups": "labs",
        "feature_pinning": "labs"
    },
    "default_federate": true,
    "default_theme": "dark",
    "roomDirectory": {
        "servers": [
            "matrix.org"
        ]
    },
    "welcomeUserId": "@riot-bot:matrix.org",
    "piwik": {
        "url": "https://piwik.riot.im/",
        "whitelistedHSUrls": ["https://matrix.org"],
        "whitelistedISUrls": ["https://vector.im", "https://matrix.org"],
        "siteId": 1
    },
    "enable_presence_by_hs_url": {
        "https://matrix.org": false
    }
}
```

Reset NGINX:

`$ sudo systemctl restart nginx`

You should be able to view and use Riot-Web through your URL now, test it out.

***
## Configure TURN service:

Your matrix server still cannot make calls across NATs (different routers), for this we need to configure coturn.

Configure a simple A DNS record pointing turn.example.org to your servers IP.

`$ sudo apt install coturn`

Generate a ‘shared-secret-key’, this can be done like so:
```
$ < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c${1:-64};echo;
5PFhfL1Eoe8Wa6WUxpR4wcwKUqkcl3UUg9QeOmpfnGHpW2O9cOsZ5yIoCDgMMdVP
```

Edited coturn configs:
```
$ sudo nano /etc/turnserver.conf

lt-cred-mech
use-auth-secret
static-auth-secret=shared-secret-key
realm=turn.example.org
no-tcp-relay
allowed-peer-ip=10.0.0.1
user-quota=16
total-quota=1200
min-port=49152
max-port=65535

$ sudo nano /etc/default/coturn

#
# Uncomment it if you want to have the turnserver running as
# an automatic system service daemon
#
TURNSERVER_ENABLED=1
```

Edit homeserver.yaml:
```
$ sudo nano /etc/matrix-synapse/homeserver.yaml

turn_uris: [ "turn:turn.example.org:3478?transport=udp", "turn:turn.example.org:3478?transport=tcp" ]
turn_shared_secret: shared-secret-key
turn_user_lifetime: 86400000
turn_allow_guests: True
```
Restart both coturn and matrix-synapse and test:
```
$ sudo systemctl start coturn
$ sudo systemctl restart matrix-synapse
```
***
Now that Synapse is shutdown and we can login to matrix-synapse user:

```
$ sudo -u matrix-synapse -s /bin/bash
$ cd
```

You should land immediately to matrix-synapse's home directory which is /var/lib/matrix-synapse. Typing cd anytime brings you back here.

***
## Done!

Your Synapse is now up and running and your hosting the latest Riot through the Nginx web server.

You should familiarise yourself with the Synapse wiki: https://github.com/matrix-org/synapse/wiki

***
## Extra - Jitsi Setup

Jitsi is the usual conferencing software used with Matrix instances, hosting your own might give you reduced latency.

First create a new  A record for jitsi.example.org

Expand SSL cert so jitsi.example.org is a known subdomain:
```
$ sudo service nginx stop
$ sudo certbot certonly --rsa-key-size 4096 -d example.org -d jitsi.example.org
Press (1) then (e)
$ sudo service nginx start
```
Then install jitsi:
```
$ sudo nano /etc/apt/sources.list.d/jitsi-unstable.list
add: 'deb https://download.jitsi.org unstable/'
$ wget -qO -  https://download.jitsi.org/jitsi-key.gpg.key | sudo apt-key add -
$ sudo apt update
$ sudo apt -y install jitsi-meet
```
When prompted for hostname of the current installation for jisti-videobridge2:
- enter 'jitsi.example.org'
- enter '/etc/letsencrypt/live/example.org/privkey.pem'
- enter '/etc/letsencrypt/live/example.org/fullchain.pem'

Edited jitsi nginx conf like so:

```
$ sudo nano /etc/nginx/sites-enabled/jitsi.example.org.conf

server_names_hash_bucket_size 64;

server {
    listen 80;
    listen [::]:80;
    server_name jitsi.example.org;

    location ^~ /.well-known/acme-challenge/ {
       default_type "text/plain";
       root         /usr/share/jitsi-meet;
    }
    location = /.well-known/acme-challenge/ {
       return 404;
    }
    location / {
       return 301 https://$host$request_uri;
    }
}
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name jitsi.example.org;

    ssl_session_cache   shared:NGX_SSL_CACHE:10m;
    ssl_session_timeout 12h;
    ssl_protocols TLSv1.3 TLSv1.2;
    ssl_prefer_server_ciphers on;
    ssl_ciphers "TLS13-CHACHA20-POLY1305-SHA256:TLS13-AES-256-GCM-SHA384:TLS13-AES-128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES128-GCM-SHA256";
    ssl_dhparam         /etc/letsencrypt/live/example.org/dhparam4096.pem;
    ssl_ecdh_curve      X25519:secp521r1:secp384r1:prime256v1;

    add_header Strict-Transport-Security "max-age=15552000";
    add_header X-Content-Type-Options "nosniff" always;

    ssl_certificate /etc/letsencrypt/live/example.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.org/privkey.pem;

    root /usr/share/jitsi-meet;

    # ssi on with javascript for multidomain variables in config.js
    ssi on;
    ssi_types application/x-javascript application/javascript;

    index index.html index.htm;
    error_page 404 /static/404.html;

    gzip on;
    gzip_types text/plain text/css application/javascript application/json;
    gzip_vary on;

    location = /config.js {
        alias /etc/jitsi/meet/jitsi.example.org-config.js;
    }

    location = /external_api.js {
        alias /usr/share/jitsi-meet/libs/external_api.min.js;
    }

    #ensure all static content can always be found first
    location ~ ^/(libs|css|static|images|fonts|lang|sounds|connection_optimization|.well-known)/(.*)$
    {
        add_header 'Access-Control-Allow-Origin' '*';
        alias /usr/share/jitsi-meet/$1/$2;
    }

    # BOSH
    location = /http-bind {
        proxy_pass      http://localhost:5280/http-bind;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header Host $http_host;
    }

    # xmpp websockets
    location = /xmpp-websocket {
        proxy_pass http://127.0.0.1:5280/xmpp-websocket?prefix=$prefix&$args;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $http_host;
        tcp_nodelay on;
    }

    location ~ ^/([^/?&:'"]+)$ {
        try_files $uri @root_path;
    }

    location @root_path {
        rewrite ^/(.*)$ / break;
    }

    location ~ ^/([^/?&:'"]+)/config.js$
    {
       set $subdomain "$1.";
       set $subdir "$1/";

       alias /etc/jitsi/meet/jitsi.example.org-config.js;
    }

    #Anything that didn't match above, and isn't a real file, assume it's a room name and redirect to /
    location ~ ^/([^/?&:'"]+)/(.*)$ {
        set $subdomain "$1.";
        set $subdir "$1/";
        rewrite ^/([^/?&:'"]+)/(.*)$ /$2;
    }

    # BOSH for subdomains
    location ~ ^/([^/?&:'"]+)/http-bind {
        set $subdomain "$1.";
        set $subdir "$1/";
        set $prefix "$1";

        rewrite ^/(.*)$ /http-bind;
    }

    # websockets for subdomains
    location ~ ^/([^/?&:'"]+)/xmpp-websocket {
        set $subdomain "$1.";
        set $subdir "$1/";
        set $prefix "$1";

        rewrite ^/(.*)$ /xmpp-websocket;
    }
}
```

Inspect jitsi.example.org, if it isn't showing jitsi restart these services:

```$ sudo service jitsi-videobridge2 restart
$ sudo service nginx restart```
