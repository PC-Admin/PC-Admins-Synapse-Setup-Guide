
# PC-Admin's Synapse Setup Guide


This guide covers complete Synapse setup for Debian 9 with Postgresql. It includes the often missing sections on how to configure postgresql and coturn with Synapse. You can use this guide to make your own encrypted chat server.

You will need at least a 1GB VPS although I recommend 2GB. You will also need a desired domain name. My guide will use ‘yourserver.org’ with Riot-Web hosted through NGINX on the same server. You may wish to have your matrix service hosted at another prefix like ‘matrix.yourserver.org’.

Contact me at: @PC-Admin:perthchat.org if you get stuck or have an edit in mind.
***
## Licensing

This work is licensed under Creative Commons Attribution Share Alike 4.0, for more information on this license see here: https://creativecommons.org/licenses/by-sa/4.0/
***
## Server Setup

Configure a Debian 9 server with auto-updates, security and SSH access.
***

## DNS Records

Set up a simple A record. With ‘yourserver.org’ pointed to your servers IP. Additionally you might setup a DNS SRV record, though it's only necessary, when you changed your federation port to listen on another port the the default port 8448.

Example DNS SRV record: _matrix._tcp        3600 IN SRV     10 0 8448 yourserver.org

***

## Prepare Server

`$ sudo apt install -y apt-transport-https lsof curl python`

Inside /etc/apt/sources.list.d/matrix.list, add the following two lines:
```
deb https://matrix.org/packages/debian/ stretch main
deb-src https://matrix.org/packages/debian/ stretch main
```
`$ sudo nano /etc/apt/sources.list.d/matrix.list`
***
## Installing Matrix

`$ wget https://matrix.org/packages/debian/repo-key.asc`

`$ sudo apt-key add repo-key.asc`

`$ sudo apt update && sudo apt upgrade && sudo apt autoremove`

`$ sudo apt install matrix-synapse -y`

Asked to set name of your server, enter your desired URL here. (eg: yourserver.org)

Finally check that the synapse server is shutdown

`$ systemctl stop matrix-synapse`

In case you want to activate URL previews you need to additionally add python-lxml:

`$ sudo apt install python-lxml -y`

***
## Installing Postgresql
The default synapse install generates a config that uses sqlite. It has the advantage of being easy to setup as there's no db server setup to take care about. But from my experience the performance penalty is quite big and if you want to do something more then testing or running a small non federated server, switching to postgres should be a mandatory step.

So let's install postgresql and python driver:
`$ sudo apt install postgresql postgresql-client python-psycopg2`

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

Exit from psql by typing `\q` 

All done. Let's exit from postgres account by typing exit so land back at our own user.

***
## Adopt Synapse config to use Postgresql
Now as we have created the db and a user to be able to connect, we need to change the synapse config to use it:
Open /etc/matrix-synapse/home.server.yaml
Before the change it should look like
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

That needs to be adopted and should look like:
```
database:
    name: psycopg2
    args:
        user: synapse
        password: <your db user password>
        database: synapse
        host: localhost
        cp_min: 5
        cp_max: 10
```

Now synapse should be ready and we can see if it starts without errors:

`$ systemctl start matrix-synapse`

***

## Certbot Setup

`$ sudo apt install certbot`

Test if server IP can be pinged first, if it can then run:

`$ sudo certbot certonly`

Choose ‘spin up a temporary webserver’

enter a recovery email

enter ‘yourserver.org’ as the domain

```
Generating key (2048 bits): /etc/letsencrypt/keys/0000_key-certbot.pem
Creating CSR: /etc/letsencrypt/csr/0000_csr-certbot.pem

IMPORTANT NOTES:
 - Congratulations! Your certificate and chain have been saved at
   /etc/letsencrypt/live/yourserver.org/fullchain.pem. Your cert will
   expire on 2017-11-01. 
```
***

## Setup SSL Auto-renewal

for monthly renewal, set a crontab:

`$ sudo crontab -e`

Insert Line:

`@monthly certbot renew --quiet --post-hook "systemctl reload nginx"`

^ If this doesn’t work you're experiencing the Debian 9 bug i noticed where letsencrypt doesn't want to renew while nginx is on. Here is how i automated it:

`$ sudo /home/username/letsencrypt-record`

`$ sudo chmod 644 /home/username/letsencrypt-record`

`$ sudo touch /root/certbot-update.sh`

`$ sudo chmod 744 /root/certbot-update.sh`

`$ sudo nano /root/certbot-update.sh`

Then edit in:
```
#!/bin/bash

service nginx stop
certbot renew --quiet
service nginx start

now=$(date +"%m_%d_%Y")
echo SSL updated on: $now >> /home/username/letsencrypt-record
```

$ sudo crontab -e
```
## SSL Renewal
01 00 01 * * /bin/sh /root/certbot-update.sh
```

***

## Configure NGINX with A+ SSL

Generate dhparam key and move it to your letsencrypt folder:
```
$ openssl dhparam -out dhparam2048.pem 2048
$ sudo cp ./dhparam2048.pem /etc/letsencrypt/live/yourserver.org
```
Install NGINX and configure:
```
$ sudo apt install nginx -y
$ sudo nano /etc/nginx/conf.d/matrix.conf
```
Add:

```
server {
       listen         80;
       server_name    yourserver.org;
       return         301 https://$server_name$request_uri;
}

server {
    listen 443 ssl;
    server_name yourserver.org;

    ssl_certificate     /etc/letsencrypt/live/yourserver.org/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourserver.org/privkey.pem;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers		'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-AES128-SHA256:ECDHE-RSA-AES128-SHA256:ECDHE-ECDSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES128-SHA:ECDHE-ECDSA-AES256-SHA384:ECDHE-ECDSA-AES256-SHA:ECDHE-RSA-AES256-SHA:DHE-RSA-AES128-SHA256:DHE-RSA-AES128-SHA:DHE-RSA-AES256-SHA256:DHE-RSA-AES256-SHA:ECDHE-ECDSA-DES-CBC3-SHA:ECDHE-RSA-DES-CBC3-SHA:EDH-RSA-DES-CBC3-SHA:AES128-GCM-SHA256:AES256-GCM-SHA384:AES128-SHA256:AES256-SHA256:AES128-SHA:AES256-SHA:DES-CBC3-SHA:!DSS';
    ssl_dhparam         /etc/letsencrypt/live/yourserver.org/dhparam2048.pem;
    ssl_ecdh_curve      secp384r1;
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

    location /_matrix {
        proxy_pass http://127.0.0.1:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

^ Make sure to replace the server name here with yours!

Restart service and renew SSL:
```
$ sudo service nginx stop
$ sudo certbot renew
$ sudo service nginx start
```
***

## Fine Tune Synapse
There're two files that manage the behaviour of synapse:
 
- Server config file in /etc/matrix-synapse/homeserver.yaml
- Env file in /etc/default/matrix-synapse

The first is used to do the configuration of synapse, the second is used to setup the environement synapse is running in. Some ENV variables have an effect on the configuration.

### Registration and guest access
- Registration

    File:  /etc/matrix-synapse/homeserver.yaml: **enable_registration: True**
    
- Guest Access

    File: /etc/matrix-synapse/homeserver.yaml: **allow_guest_access: True**

**There are other settings here you may want to adjust. I would do so one at a time, testing each change as you go.**

### Cache factor

For a small server (<=2GB), an adoption of the cache factor might improve performance. Some time ago the advice was to reduce the cache factor to use less RAM. Experience has shown that the effect is quite the opposite, see [Issue](https://github.com/matrix-org/synapse/pull/4276).

So the new advice is to raise the cache factor instead, with a value of 2 being a good starting point:

- Cache factor

    File: /etc/default/matrix-synapse: SYNAPSE_CACHE_FACTOR=2.0
 
***

<span style="color:red">**Don't forget to restart synapse and examine the RAM usage after each change:**</span>

`$ sudo service matrix-synapse restart`
***

## Load Riot-Web client into NGINX

NGINX content location: /usr/share/nginx/html/index.html

https://github.com/vector-im/riot-web/releases/latest

Download latest riot-web and move contents into nginx folder:
```
~/riot-web$ wget https://github.com/vector-im/riot-web/releases/download/v0.11.4/riot-v0.11.4.tar.gz
$ tar -zxvf ./riot-v0.11.4.tar.gz
$ sudo rm -r /usr/share/nginx/html/*
$ sudo mv ./riot-v0.11.4/* /usr/share/nginx/html/
```
Create and edit config.json in nginx directory:
```
$ sudo cp /usr/share/nginx/html/config.sample.json /usr/share/nginx/html/config.json

$ sudo nano /usr/share/nginx/html/config.json

{
    "default_hs_url": "https://yourserver.org",
    "default_is_url": "https://vector.im",
    "brand": "Riot",
    "integrations_ui_url": "https://scalar.vector.im/",
    "integrations_rest_url": "https://scalar.vector.im/api",
    "bug_report_endpoint_url": "https://riot.im/bugreports/submit",
    "enableLabs": true,
    "roomDirectory": {
        "servers": [
            "matrix.org"
        ]
    },
    "welcomeUserId": "@riot-bot:matrix.org",
    "piwik": {
        "url": "https://piwik.riot.im/",
        "siteId": 1
    }
}
```
Reset NGINX:

`$ sudo systemctl restart nginx`

You should be able to view and use Riot-Web through your URL now, test it out.
***

## Configure TURN service:

Your matrix server still cannot make calls across NATs (different routers), for this we need to configure coturn.

Configure a simple A DNS record pointing turn.yourserver.org to your servers IP.

`$ sudo apt install coturn`

Generate a ‘shared-secret-key’, this can be done like so:
```
$ < /dev/urandom tr -dc _A-Z-a-z-0-9 head -c64
V2OuWAio2B8sBpIt6vJk8Hmv1FRapQJDmNhhDEqjZf0mCyyIlOpf3PtWNT6WfWSh
```

Edited coturn configs:
```
$ sudo nano /etc/turnserver.conf

lt-cred-mech
use-auth-secret
static-auth-secret=[shared-secret-key]
realm=turn.yourserver.org
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

turn_uris: [ "turn:turn.yourserver.org:3478?transport=udp", "turn:turn.yourserver.org:3478?transport=tcp" ]
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

**Done!**
***

Now your server is up and running consider registering on the hello-matrix list of servers: https://www.hello-matrix.net/public_servers.php or at https://matrix.to/#/#hello-matrix:matrix.org
