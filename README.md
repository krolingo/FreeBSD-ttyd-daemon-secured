# Securing [ttyd](https://github.com/tsl0922/ttyd)

## Terminal over the Web, Secured

Following the "[FreeBSD: how to create a (rc) system service](https://youtu.be/_4NhXojyRx8?si=BRZ73lXlIkiGj9jp)" tutorial from the **@BSDJedi** "[Cool Bits and Such](https://youtube.com/@bsdjedi?si=Q3juFPOTn8Ffop1s)" YT channel, I decided to take it further by making ttyd secure as I know how. I'm no expert, so there is that. This is the result of my efforts in trying to secure ttyd as best as I could.

### In this setup:

* `ttyd` is secured with a self-signed certificate.
* Remote access to`ttyd` is secured through a reverse proxy that utilizes a real SSL certificate.
* The credentials required for`ttyd` authentication are stored in a secure environment file located at`/etc/ttyd_env`.
* A wrapper script named`ttyd_wrapper.sh` reads the credentials from`/etc/ttyd_env` and executes`ttyd` with these credentials.
* The main`ttyd` service script utilizes`daemon` to execute the wrapper script (`ttyd_wrapper.sh`). This approach ensures that the credentials used for authentication are not visible in the process list (`ps` output).
* By implementing this setup, the credentials remain securely stored and inaccessible from unauthorized access, enhancing the overall security of the`ttyd` service.

## Step 1: Create Certificates Directory for ttyd

Create the directory for storing SSL private keys:

```sh
mkdir /usr/local/etc/ssl/private
```

## Step 2: Generate SSL Certificates

### Generate a Private Key

```sh
openssl genpkey -algorithm RSA -out /usr/local/etc/ssl/private/ttyd.key
```

### Generate a Certificate Signing Request (CSR)

```sh
openssl req -new -key /usr/local/etc/ssl/private/ttyd.key -out /usr/local/etc/ssl/private/ttyd.csr
```

Follow the prompts to enter your information.

### Generate a Self-Signed Certificate

```sh
openssl x509 -req -days 365 -in /usr/local/etc/ssl/private/ttyd.csr -signkey /usr/local/etc/ssl/private/ttyd.key -out /usr/local/etc/ssl/certs/ttyd.crt
```

## Step 3: Modify the rc.d Script

Edit your rc.d script to include the SSL options and paths to the certificate files. Here is the updated script:

```sh
cat /usr/local/etc/rc.d/ttyd
```

```sh
#!/bin/sh
#

# PROVIDE: ttyd

# REQUIRE: netif

# BEFORE:

# KEYWORD: shutdown

. /etc/rc.subr

name="ttyd"
desc="ttyd xterm on browser service"
rcvar="ttyd_enable"
log_file="/var/log/${name}.log"
pidfile="/var/run/${name}.pid"
ttyd_exec="/usr/local/bin/ttyd_wrapper.sh"
ttyd_prog="/usr/local/bin/zsh"
ttyd_user="YOUR_USER_ACCOUNT_NAME"
daemon="/usr/sbin/daemon"
daemon_opt="-r -u ${ttyd_user} -P ${pidfile}"

load_rc_config "$name"
: ${ttyd_enable="NO"}

start_cmd="ttyd_start"
stop_cmd="ttyd_stop"

ttyd_start() {
if [ -f "${pidfile}" ]; then
echo "ttyd: service is already running!"
else
${daemon} ${daemon_opt} ${ttyd_exec} ${ttyd_prog} > "${log_file}" 2>&1
fi
}

ttyd_stop() {
if [ -f "${pidfile}" ]; then
pid=$(cat "${pidfile}")
kill "${pid}" && rm -f "${pidfile}"
else
echo "ttyd: service is not running!"
fi
}

run_rc_command "$1"
```

## Step 4: Secure the Password

Create an Environment File to Store the Credentials

Create a file, e.g., /usr/local/etc/ttyd_env, and add the environment variables:

```sh
cat /usr/local/etc/ttyd_env
TTYD_USER="YOUR_USER_ACCOUNT_NAME"
TTYD_PASS="YOUR_PASSWORD"
```

Set the appropriate permissions and ownership:

```sh
chmod 640 /usr/local/etc/ttyd_env
chown root:ttyd /usr/local/etc/ttyd_env
```

## Step 5: Create a Wrapper Script

```sh
cat /usr/local/bin/ttyd_wrapper.sh
```

```sh
#!/bin/sh

# Source the environment file with credentials

. /usr/local/etc/ttyd_env

# Run ttyd with the credentials from the environment variables

exec /usr/local/bin/ttyd --writable -p 7682 -c "${TTYD_USER}:${TTYD_PASS}" -w /home/${TTYD_USER} -o --ssl --ssl-cert /usr/local/etc/ssl/certs/ttyd.crt --ssl-key /usr/local/etc/ssl/private/ttyd.key "$@"
```

Make the wrapper script executable:

```sh
chmod +x /usr/local/bin/ttyd_wrapper.sh
```

## Step 6: Create Certificate for Nginx

To set up a proper proxy for ttyd, generate a real certificate to avoid browser warnings and enhance security.

```sh
pkg install py311-certbot-dns-cloudflare
mkdir -p /root/.secrets/certbot/
nano /root/.secrets/certbot/cloudflare.ini
```

Add the following to the cloudflare.ini file:

```sh
dns_cloudflare_api_token = "YOUR_API_TOKEN"
```

Set the appropriate permissions:

```sh
chmod -R 600 /root/.secrets
chmod 0400 /root/.secrets/certbot/cloudflare.ini
```

Obtain the certificate:

```sh
certbot certonly --dns-cloudflare --dns-cloudflare-credentials ~/.secrets/certbot/cloudflare.ini -d server.domain.com
```

## Step 7: Install and Configure Nginx as Reverse Proxy

Install Nginx

```sh
doas pkg install nginx
```

### Configure Nginx as Reverse Proxy

Edit your Nginx configuration to set up the reverse proxy. Here’s a basic configuration example:

```sh
cat /usr/local/etc/nginx/conf.d/server.domain.com.conf
```

```sh
server {
listen 80;
server_name server.domain.com;

location / {
return 301 https://$host$request_uri;
 }
}

server {
listen 443 ssl;
server_name server.domain.com;

ssl_certificate /usr/local/etc/letsencrypt/live/server.domain.com/fullchain.pem;
ssl_certificate_key /usr/local/etc/letsencrypt/live/server.domain.com/privkey.pem;

ssl_protocols TLSv1.2 TLSv1.3;
ssl_ciphers HIGH:!aNULL:!MD5;

location ~ ^/ttyd(.*)$ {
proxy_http_version 1.1;
proxy_set_header Host $host;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
proxy_set_header Upgrade $http_upgrade;
proxy_set_header Connection "upgrade";
proxy_pass https://127.0.0.1:7682/$1;
 }
}
```
