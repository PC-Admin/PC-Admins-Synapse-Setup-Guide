
**PC-Admin's Synapse Setup Guide**

.. contents::

This guide covers Synapse setup for Debian 9. It includes the often missing sections on how to configure postgresql and coturn with Synapse. You can use this guide to make your own encrypted chat server.

You will need at least a 1GB VPS although I recommend 2GB. You will also need a desired domain name. My guide will use ‘yourserver.org’ with Riot-Web hosted through NGINX on the same server. You may wish to have your matrix service hosted at another prefix like ‘matrix.yourserver.org’.


Server Setup
============

Configure a Debian 9 server with auto-updates, security and SSH access.


DNS Records
===========

Set up a simple A record. With ‘yourserver.org’ pointed to your servers IP.


Prepare Server
==============

$ sudo apt install -y apt-transport-https lsof curl python python-pip

| Inside /etc/apt/sources.list.d/matrix.list, add the following two lines:
| 	deb https://matrix.org/packages/debian/ stretch main
| 	deb-src https://matrix.org/packages/debian/ stretch main
| 
| $ sudo nano /etc/apt/sources.list.d/matrix.list


Installing Matrix
=================

$ wget https://matrix.org/packages/debian/repo-key.asc | sudo apt-key add -

$ sudo apt update && sudo apt upgrade && sudo apt autoremove

$ sudo apt install matrix-synapse -y

Asked to set name of server: ‘yourserver.org’


Configure Firewall
==================

Open the following ports:

| $ sudo ufw allow 443
| $ sudo ufw allow 8448
| $ sudo ufw allow 80
| 
| If you have an external firewall, open these ports there.


Certbot Setup
=============

$ sudo apt install certbot

Test if server IP can be pinged first, if it can then run:

$ sudo certbot certonly

| choose ‘spin up a temporary webserver’
| enter a recovery email
| enter ‘yourserver.org’ as the domain
| 
| Generating key (2048 bits): /etc/letsencrypt/keys/0000_key-certbot.pem
| Creating CSR: /etc/letsencrypt/csr/0000_csr-certbot.pem
| 
| IMPORTANT NOTES:
|  - Congratulations! Your certificate and chain have been saved at
|    /etc/letsencrypt/live/yourserver.org/fullchain.pem. Your cert will
|    expire on 2017-11-01. 
| 
| for 3 month renewal, set a crontab:
| 
| $ sudo crontab -e
| 
| Insert Line:
| @monthly certbot renew --quiet --post-hook "systemctl reload nginx"
| 
| ^ This doesn’t work. If anyone has the solution for renewal please contact me.
| 
| $ sudo ls /etc/letsencrypt/live/yourserver.org
| cert.pem  chain.pem  fullchain.pem  privkey.pem  README


Configure NGINX with A+ SSL
===========================

| Generate dhparam key and move it to your letsencrypt folder:
| $ openssl dhparam -out dhparam2048.pem 2048
| $ sudo cp ./dhparam2048.pem /etc/letsencrypt/live/yourserver.org
| 
| $ sudo apt install nginx -y
| 
| $ sudo nano /etc/nginx/conf.d/matrix.conf
| 
| Add:
| 
| server {
|        listen         80;
|        server_name    yourserver.org;
|        return         301 https://$server_name$request_uri;
| }
| 
| server {
|     listen 443 ssl;
|     server_name yourserver.org;
| 
|     ssl_certificate     /etc/letsencrypt/live/yourserver.org/fullchain.pem;
|     ssl_certificate_key /etc/letsencrypt/live/yourserver.org/privkey.pem;
|     ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
|     ssl_ciphers         'ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDHE-ECDSA-AES1$
|     ssl_dhparam         /etc/letsencrypt/live/yourserver.org/dhparam2048.pem;
|     ssl_ecdh_curve      secp384r1;
|     add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;
| 
|     location /_matrix {
|         proxy_pass http://127.0.0.1:8008;
|         proxy_set_header X-Forwarded-For $remote_addr;
|     }
| }
| 
| Make sure to replace the server name here!
| 
| Restart service and renew SSL:
| $ sudo service nginx stop
| 
| $ sudo certbot renew
| 
| $ sudo service nginx start


Fine Tune Synapse
=================

Edit /etc/matrix-synapse/homeserver.yaml:

| # A list of other Home Servers to fetch the public room directory from
| # and include in the public room directory of this home server
| # This is a temporary stopgap solution to populate new server with a
| # list of rooms until there exists a good solution of a decentralized
| # room directory.
| secondary_directory_servers:
|     - matrix.org
|     - vector.im
| 
| If you want you can also:
| 
| Enable Self Registration
| 
| $ sudo nano /etc/matrix-synapse/homeserver.yaml
| enable_registration: True
| 
| Allow Guests
| 
| # Allows users to register as guests without a password/email/etc, and
| # participate in rooms hosted on this server which have been made
| # accessible to anonymous users.
| allow_guest_access: True
| 
| There are other settings here you may want to adjust. I would do so one at a time with testing.
| 
| Also check environmental variables in /etc/default/matrix-synapse for a small server (<=2GB), you will | want to edit in a low cache factor:
| 
| # Specify environment variables used when running Synapse
| # SYNAPSE_CACHE_FACTOR=1 (default)
| 
| SYNAPSE_CACHE_FACTOR=0.05
| 
| Then restart synapse and examine the RAM usage:
| 
| $ sudo service matrix-synapse restart


Load Riot-Web client into NGINX
===============================

| NGINX content location:
| /usr/share/nginx/html/index.html
| 
| https://github.com/vector-im/riot-web/releases/latest
| 
| ~/riot-web$ wget https://github.com/vector-im/riot-web/releases/download/v0.11.4/riot-v0.11.4.tar.gz
| $ tar -zxvf ./riot-v0.11.4.tar.gz
| $ sudo rm -r /usr/share/nginx/html/*
| $ sudo mv ./riot-v0.11.4/* /usr/share/nginx/html/
| 
| Nope… reset nginx?
| 
| $ sudo systemctl restart nginx
| 
| You should be able to view and use Riot-web through your URL now, test it out.


Configure TURN service:
=======================

Your matrix server still cannot make calls across NATs, for this we need to configure coturn.

Configure a simple A DNS record pointing turn.yourserver.org to your servers IP.

$ sudo apt install coturn

| Generate a ‘shared-secret-key’, this can be done like so:
| $ < /dev/urandom tr -dc _A-Z-a-z-0-9 | head -c64
| V2OuWAio2B8sBpIt6vJk8Hmv1FRapQJDmNhhDEqjZf0mCyyIlOpf3PtWNT6WfWSh
| 
| $ sudo nano /etc/turnserver.conf
| Edited so that:
| lt-cred-mech
| use-auth-secret
| static-auth-secret=[shared-secret-key]
| realm=turn.yourserver.org
| no-tcp-relay
| allowed-peer-ip=10.0.0.1
| user-quota=16
| total-quota=1200
| min-port=49152
| max-port=65535
| 
| $ sudo nano /etc/default/coturn
| #
| # Uncomment it if you want to have the turnserver running as
| # an automatic system service daemon
| #
| TURNSERVER_ENABLED=1
| 
| $ sudo ufw allow 3478
| 
| $ sudo nano /etc/matrix-synapse/homeserver.yaml
| turn_uris: [ "turn:turn.yourserver.org:3478?transport=udp", "turn:turn.yourserver.org:3478?transport=tcp" ]
| turn_shared_secret: shared-secret-key
| turn_user_lifetime: 86400000
| turn_allow_guests: True
| 
| $ sudo systemctl start coturn
| 
| $ sudo systemctl restart matrix-synapse


Configure PostgreSQL database
=============================

By default synapse uses a sqlite3 database, performance and scalability is greatly improved by changing over to a PostgreSQL database. If you plan to ever have more than ~20 users I would recommend this.

| Install PostgreSQL
| $ sudo apt install postgresql libpq-dev postgresql-client postgresql-client-common
| 
| 
| Create Role and Database
| $ sudo -i -u postgres
| 
| $ createuser synapse -P --interactive
| 
| postgres@VM:~$ createuser synapse -P --interactive
| Enter password for new role: 
| Enter it again: 
| Shall the new role be a superuser? (y/n) n
| Shall the new role be allowed to create databases? (y/n) y
| Shall the new role be allowed to create more new roles? (y/n) y
| 
| Now we're back at $postgres. Let's create a database for Synapse with correct settings and set the owner to be the user we just created:
| 
| Type: psql
| ..And create the database as follows:
| postgres=# CREATE DATABASE synapse WITH ENCODING 'UTF8' LC_COLLATE 'C' LC_CTYPE 'C' TEMPLATE template0 OWNER synapse; 
| 
| Exit from psql by typing \q 
| 
| All done. Let's exit from postgres account by typing exit so land back at our own user.
| 
| 
| Next we modify postgres pg_hba.conf to allow all connections from localhost to the local database server:
| $ sudo nano /etc/postgresql/9.6/main/pg_hba.conf
| !NOTE "Paste it under the "Put your actual configuration here"
| host all all 127.0.0.1/32 trust
| 
| Restart postgresql after the change:
| $ sudo service postgresql restart
| 
| Shutdown matrix-synapse for now:
| $ sudo service matrix-synapse stop 
| 
| Let's give the user ‘matrix-synapse’ access to bash temporary so we login to it's shell. The port process felt easier when I can actually work with the synapse user (python/envs/permissions work nicely) We will undo this change later:
| 
| $ sudoedit /etc/passwd
| !NOTE, I use "sudoedit" by habit but you could also use "sudo nano /etc/passwd" so it's up your preference.
| Change the shell for user matrix-synapse from /bin/false to /bin/bash, it's at the end of the row:
| matrix-synapse:x:XXX:XXXXX::/var/lib/matrix-synapse:/bin/bash
| 
| Now that Synapse is shutdown and we can login to matrix-synapse user:
| $ sudo -i -u matrix-synapse
| You should land immediately to matrix-synapse's home directory which is /var/lib/matrix-synapse. Typing cd anytime brings you back here.
| 
| Install psycopg2:
| $ pip install psycopg2
| !NOTE Ignore any traceback errors if you get and no use to try sudo as this is not an admin user
| 
| 
| You should land immediately to matrix-synapse's home directory which is /var/lib/matrix-synapse. Typing cd anytime brings you back here. This location has the original SQLite homeserver.db, which we want to snapshot(copy) now, when Synapse is turned off. Let's take a snapshot:
| $ cp homeserver.db homeserver.db.snapshot
| !NOTE, no need to use sudo anytime when you are logged in as matrix-synapse. This user is not an admin(in sudoers file) and it already has correct permissions for the needed files/db's/directories's. 
| 
| $ ls
| homeserver.db  media  uploads
| 
| Restart service for now:
| $ exit
| $ sudo service matrix-synapse start
| 
| Login back to matrix-synapse account:
| $ sudo -i -u matrix-synapse
| Make a copy of the homeserver.yaml configuration file to be modified for our postgresql database settings::
| $ cp /etc/matrix-synapse/homeserver.yaml /etc/matrix-synapse/homeserver-postgres.yaml
| Modify the postgres database settings to the new homeserver-postgres.yaml -file:
| $ nano /etc/matrix-synapse/homeserver-postgres.yaml
| Fill in the database section as follows:
| database:
|     name: psycopg2
|     args:
|         user: synapse
|         password: YOUR_SICK_DB_PASSWORD_PLEASE_SAVE_THIS_SOMEWHERE
|         database: synapse
|         host: localhost
|         cp_min: 5
|         cp_max: 10
| !NOTE user,password,database are the values we created with psql before.
| 
| 
| Download synapse_port_db.py:
| 
| https://github.com/matrix-org/synapse/blob/master/scripts/synapse_port_db
| Set excecute permissions to the synapse_port_db.py -script:
| $ chmod +x synapse_port_db.py
| 
| Now we are ready to try the port script against the homeserver.db.snapshot:
| $ python synapse_port_db.py --sqlite-database homeserver.db.snapshot --postgres-config /etc/matrix-synapse/homeserver-postgres.yaml --curses -v
| This should run a long time if you've used SQLite DB for a while. The --curses and -v flags at the end help you visualize what's going on. It will show you in real time what data is migrated from the homeserver.db.snapshot to your new postgresql database. At the end the screen should be pretty much all green (I think I had like 2 "events" missing. Press any key..
| Almost at the finale. To complete the conversion shut down the synapse server and run the port script one last time, e.g. if the SQLite database is at homeserver.db:
| Move back to your normal user account (eg. exit from matrix-synapse):
| exit
| $ sudo service matrix-synapse stop
| Change user back to matrix-synapse:
| $ sudo -i -u matrix-synapse
| And let's run the portscript again to bring the latest changes to postgresql:
| python synapse_port_db.py --sqlite-database homeserver.db --postgres-config /etc/matrix-synapse/homeserver-postgres.yaml --curses -v
| This shouldn't take so long as it quickly figures to import incrementally (e.g) only the data that has changed during Synapse was up.
| 
| 
| Last step is to rename our new homeserver-postgresql.yaml to homeserver.yaml
| e.g:
| $ cd /etc/matrix-synapse/
| $ mv homeserver.yaml homeserver.yaml.old
| $ mv homeserver-postgres.yaml homeserver.yaml
| * And restart Synapse *
| $ exit from matrix-synapse -user
| $ sudo service matrix-synapse start
| Synapse should now be running against PostgreSQL, Wohoo!
| * Final thing is to deny shell from matrix-synapse, like it was before*:
| $ sudoedit /etc/passwd
| matrix-synapse:x:XXX:XXXXX::/var/lib/matrix-synapse:/bin/*false*
| 
| Done! :)
| 
| 
| Cleanup these old files after testing:
| 
| /etc/matrix-synapse/homeserver.yaml.old 
| /var/lib/matrix-synapse/homeserver.db  
| /var/lib/matrix-synapse/homeserver.db.snapshot 
| /var/lib/matrix-synapse/port-synapse.log 
| /var/lib/matrix-synapse/synapse_port_db.py 



