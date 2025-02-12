# Matrix
https://upcloud.com/community/tutorials/install-matrix-synapse/

Install packages
```
sudo apt-get install build-essential python3-dev libffi-dev python-pip python-setuptools sqlite3 libssl-dev python-virtualenv libjpeg-dev libxslt1-dev
```

Installing Synapse home server
```
mkdir -p ~/synapse
virtualenv -p python3 ~/synapse/env
source ~/synapse/env/bin/activate
```

Check that the following packages are installed and up to date.
```
pip install --upgrade pip virtualenv six packaging appdirs
```

Then use pip to install and upgrade the setup tools as well as the Synapse server itself.
```
pip install --upgrade setuptools
pip install matrix-synapse
```

Running on a pi I needed to install the two following packages
```
apt-get install libopenjp2-7 libtiff5 
```

Before Synapse can be started, you will need to generate a configuration file that defines the server settings.

Run the following commands in your Synapse virtual environment. Replace the server name matrix.example.com with your own domain, but note that the name cannot be changed later without reinstalling the home server. Also, choose whether you wish to allow Synapse to report statistics by entering either --report-stats=yes or no.
```
source ~/synapse/env/bin/activate
pip install -U matrix-synapse
cd ~/synapse
python -m synapse.app.homeserver \
  --server-name matrix.example.com \
  --config-path homeserver.yaml \
  --generate-config \
  --report-stats=yes|no
  ```
  
```
A config file has been generated in 'homeserver.yaml' for server name 'matrix.example.com'. Please review this file and customise it to your needs.
```

You’ll also need to have a webserver available for Certbot to use for validating the certificate. Install nginx using the instructions below according to your operating system.
```
sudo apt-get install nginx
```

Start nginx
```
service nginx start
```

Create cert using LetsEncrypt
https://certbot.eff.org/lets-encrypt/pip-nginx
```
certbot certonly --nginx -d matrix.example.com
```

Matrix recommends setting up a reverse proxy, such as nginx, Apache or HAProxy, in front of your Synapse server. This is intended to simplify client connections by allowing Matrix to use the common HTTPS port 443 while keeping the server-to-server connections at the port 8448.

Change the default home server configuration to only listen to the localhost address for the port 8008. Open the homeserver.yaml file for edit.
```
vi ~/synapse/homeserver.yaml
```

Next, enable nginx to act as the reverse proxy by creating a configuration file for the proxy functionality.
```
vi /etc/nginx/conf.d/matrix.conf
```

Then enter the following to enable the proxy with SSL termination. Replace the matrix.example.com with your domain in the server name. The certificates issued by Let’s Encrypt are saved under the directory indicated in the Certbot output, usually under /etc/letsencrypt/live/. Again, replace the matrix.example.com with the domain name the certificates were issued for.
```
server {
    listen 80;
	listen [::]:80;
    server_name matrix.example.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    listen [::]:443 ssl;
    server_name matrix.example.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;

    location / {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}

server {
    listen 8448 ssl default_server;
    listen [::]:8448 ssl default_server;
    server_name matrix.example.com;

    ssl on;
    ssl_certificate /etc/letsencrypt/live/matrix.example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/matrix.example.com/privkey.pem;
    location / {
        proxy_pass http://localhost:8008;
        proxy_set_header X-Forwarded-For $remote_addr;
    }
}
```

Edit `/etc/nginx/nginx.conf` and uncomment
```
server_names_hash_bucket_size 64;
```

start and enable nginx.
```
sudo systemctl restart nginx
sudo systemctl enable nginx
```

Now that Synapse is installed, you should add a new user as an admin to be able to configure chat rooms and web client settings. First, make sure you are in the home server virtual environment and start the server.
```
cd ~/synapse
source env/bin/activate
synctl start
```

Then register a new user to the localhost.
```
register_new_matrix_user -c homeserver.yaml http://localhost:8008
```

This starts an interactive user configuration, enter the desired username and password, then select yes to enable administration priveledges.
```
New user localpart [root]: username
Password: password
Confirm password: password
Make admin [no]: yes|no
Sending registration request...
Success.
```

Ensure the matrix server starts on server reboot by adding a script to the crontab
```
cd ~/synapse
vi matrix_startup_script.sh
```

Paste the following in
```
#!/bin/bash
cd ~/synapse
source env/bin/activate
synctl start
```

Edit the crontab
```
crontab -e
```

Paste the following at the bottom
```
@reboot /root/synapse/start_on_boot.sh
```

# Other config
Increase the body size in nginx config by adding the following to `/etc/nginx/nginx.conf`:
```
client_max_body_size 25M;
```

Set identity server in `/root/synapse/homeserver.yaml`
```
default_identity_server: https://matrix.example.co.uk
```

Enable `search_all_users: true` in `/root/synapse/homeserver.yaml`
```
user_directory:
#  enabled: true
  search_all_users: true
```

Restart synapse
```
cd ~/synapse
source env/bin/activate
synctl start
```
