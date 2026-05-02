# Private messaging server

Below is documentation on how to set up and configure a basic self-hosted messaging server and clients, with working messaging, voice/video calls, and screen sharing.

## TLDR Setup

Setup consists of:
- a single Ubuntu VM with a fixed public IP, 1GB RAM, 1vCPU, 40GB disk,
- everything installed via Docker on that VM,
- core server: Synapse by Element (implements Matrix): https://github.com/element-hq/synapse,
- database: PostgreSQL instance: https://www.postgresql.org/,
- proxy: Caddy service: https://github.com/caddyserver/caddy,
- voice/video calls: coturn server: https://github.com/coturn/coturn,
- firewall: UFW (on Ubuntu) + network access rules (VM cloud provider).


## Installation

Steps:
- buy a domain,
- create a VM with a fixed public IP,
- install firewall, Docker,
- generate/paste/edit configs,
- copy and start Docker compose,
- create users via terminal,
- edit VM firewall (Ubuntu + cloud provider),
- install user clients (Android app, Windows, etc).

To replace in every command/config:
- `YOUR_DOMAIN` - for example `domain.com`.
- `TURN_SECRET` - calling server secret, long (32+), like `abcdef...`.
- `PG_PASSWORD` - PostgreSQL password, for example `qwertyu...`.

### Install Docker, firewall

Get updates, fish terminal, ufw firewall:
```bash
# updates
sudo apt update && sudo apt upgrade -y
# firewall ufw
sudo apt install ufw
# fish terminal
sudo apt install fish
```

Switch to fish terminal for hints:
```bash
fish
```

Install Docker, copy from:  https://docs.docker.com/engine/install/ubuntu/#install-using-the-repository or from below:
```bash
sudo apt update
sudo apt install ca-certificates curl
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc

sudo tee /etc/apt/sources.list.d/docker.sources <<EOF
Types: deb
URIs: https://download.docker.com/linux/ubuntu
Suites: $(. /etc/os-release && echo "${UBUNTU_CODENAME:-$VERSION_CODENAME}")
Components: stable
Signed-By: /etc/apt/keyrings/docker.asc
EOF

sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
```

Optionally, add current Linux user to the docker group to run 'docker ...' without sudo:
```bash
sudo usermod -aG docker $USER
```

### Create configs

Create folders:
```bash
mkdir -p ~/matrix/synapse ~/matrix/postgres ~/matrix/turn ~/matrix/caddy
cd ~/matrix
```

#### Synapse

Ensure user id and group id will not conflict with local instance users / groups - set UID and GID accordingly.

Pre-generate config:
```bash
docker run -it --rm \
  -v $(pwd)/synapse:/data \
  -e SYNAPSE_SERVER_NAME=YOUR_DOMAIN \
  -e SYNAPSE_REPORT_STATS=no \
  -e UID=946 -e GID=946 \
  matrixdotorg/synapse:v1.147.1 generate
```

Edit Synapse config:
```bash
sudo nano synapse/homeserver.yaml
```

It should look similar to:
```yaml
# Configuration file for Synapse.
#
# This is a YAML file: see [1] for a quick introduction. Note in particular
# that *indentation is important*: all the elements of a list or dictionary
# should have the same indentation.
#
# [1] https://docs.ansible.com/ansible/latest/reference_appendices/YAMLSyntax.html
#
# For more information on how to configure Synapse, including a complete accounting of        
# each option, go to docs/usage/configuration/config_documentation.md or
# https://element-hq.github.io/synapse/latest/usage/configuration/config_documentation.html   
server_name: "YOUR_DOMAIN"

# rate limiting # ADD THIS SECTION
rc_messages:
  per_second: 0.2
  burst_count: 10
rc_login:
  address:
    per_second: 0.1
    burst_count: 5

pid_file: /data/homeserver.pid
listeners:
  - port: 8008
    resources:
    - compress: false
      names:
      - client  # REMOVE FEDERATION FROM HERE 
    tls: false
    type: http
    x_forwarded: true

database: # REPLACE SQLITE WITH THE FOLLOWING
  name: psycopg2
  args:
    user: synapse_user
    password: PG_PASSWORD
    database: synapse_db
    host: postgres
    cp_min: 5
    cp_max: 10

log_config: "/data/YOUR_DOMAIN.log.config"
media_store_path: /data/media_store # here will live all uploaded photos and files
registration_shared_secret: "some_long_secret_pregenerated_is_here"
report_stats: false
macaroon_secret_key: "here_will_be_some_very_long_secret"
form_secret: "here_also_will_be_another_very_long_secret"
signing_key_path: "/data/YOUR_DOMAIN.signing.key"

trusted_key_servers: [] # one of the ways to disable federation # CHANGE HERE

# PASTE EVERYTHING WHATS BELOW

# Disable user registration - create all manually with cli
enable_registration: false
enable_registration_without_verification: false

encryption_enabled_by_default_for_room_type: all

# Disable federation entirely
federation_enabled: false
federation_domain_whitelist: []
# Do not try to discover other servers
allow_public_rooms_over_federation: false
allow_public_rooms_without_auth: false
# No public room directory
enable_room_list_search: true
# Disable URL previews (prevents server-side fetches)
url_preview_enabled: false
# Disable metrics unless needed
enable_metrics: false

# Calling support
turn_uris:
  - "turn:YOUR_DOMAIN:3478?transport=udp" # CHANGE DOMAIN
  - "turn:YOUR_DOMAIN:3478?transport=tcp"
  - "turns:YOUR_DOMAIN:5349?transport=tcp"

turn_shared_secret: "TURN_SECRET" # CHANGE TO SOME LONG STRING

turn_user_lifetime: 1h
turn_allow_guests: false

# vim:ft=yaml
```

#### Caddy

Remember to change `YOUR_DOMAIN`. Optionally, set `YOUR_EMAIL_OPTIONALLY` to your email - will be used only for ACME transactions for CA to contact you if necessary.

```bash
nano caddy/Caddyfile
```

Paste and edit:
```caddy
{
    # Global options
    email YOUR_EMAIL_OPTIONALLY
    # Disable admin API if not needed
    admin off
}

YOUR_DOMAIN {

    # Enable compression
    encode zstd gzip

    # Security headers
    header {
       X-Content-Type-Options "nosniff"
       X-Frame-Options "DENY"
       Referrer-Policy "no-referrer"
    }

    # Matrix Client-Server API
    reverse_proxy /_matrix/* homeserver:8008 {
        header_up X-Real-IP {remote_host}
        header_up X-Forwarded-For {remote_host}
        header_up X-Forwarded-Proto {scheme}
    }

    # Synapse admin / internal APIs
    reverse_proxy /_synapse/* homeserver:8008
}
```

#### coturn configs (add calls support)

Remember to change `YOUR_DOMAIN` and `TURN_SECRET`, and optionally confirm that paths to Caddy TLS certs are mounted and available at the specified paths.

```bash
nano turn/turnserver.conf
```

```conf
listening-port=3478
tls-listening-port=5349

realm=YOUR_DOMAIN
server-name=YOUR_DOMAIN

use-auth-secret
static-auth-secret=TURN_SECRET

fingerprint

min-port=49152
max-port=49200

# TLS certs from Caddy (docker volume mounts - confirm with docker compose)
cert=/certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/YOUR_DOMAIN/YOUR_DOMAIN.crt   
pkey=/certs/caddy/certificates/acme-v02.api.letsencrypt.org-directory/YOUR_DOMAIN/YOUR_DOMAIN.key   

no-cli
no-loopback-peers
no-multicast-peers

# RATE CONTROL
total-quota=10
# Bandwidth cap
bps-capacity=8388608
max-bps=16388608
# Replay protection
stale-nonce=600
```

### Copy and start Docker compose

Ensure the following values match:
- `UID=946` and `GID=946` of `homeserver` service must match values used when pre-generating Synapse config,
- `POSTGRES_PASSWORD` with `synapse/homeserver.yaml` value,


```bash
nano docker-compose.yml
```

```yaml
services:
  postgres:
    image: postgres:15.16 # updates are not necessary, but get here: https://hub.docker.com/_/postgres
    container_name: matrix-postgres
    restart: unless-stopped
    environment:
      POSTGRES_USER: synapse_user
      POSTGRES_PASSWORD: PG_PASSWORD
      POSTGRES_DB: synapse_db
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --lc-collate=C --lc-ctype=C"
    volumes:
      - ./postgres:/var/lib/postgresql/data
    networks:
      - matrix

  homeserver:
    image: matrixdotorg/synapse:v1.147.1 # check for updates: https://hub.docker.com/r/matrixdotorg/synapse
    container_name: matrix-synapse
    restart: unless-stopped
    environment:
      - UID=946
      - GID=946
    depends_on:
      - postgres
    volumes:
      - ./synapse:/data
    networks:
      - matrix

  caddy:
    image: caddy:2.11-alpine # check for updates: https://hub.docker.com/_/caddy
    container_name: matrix-caddy
    restart: unless-stopped
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./caddy/Caddyfile:/etc/caddy/Caddyfile:ro
      - ./caddy/data:/data
      - ./caddy/config:/config
    networks:
      - matrix

  turn:
    image: coturn/coturn:4.10 # check for updates: https://hub.docker.com/r/coturn/coturn
    container_name: matrix-turn
    restart: unless-stopped

    network_mode: host

    volumes:
      - ./turn/turnserver.conf:/etc/coturn/turnserver.conf:ro
      - ./turn/data:/var/lib/coturn
      - ./caddy/data:/certs:ro # this mounts Caddy TLS certificates

    # Logging to stdout
    command: >
      -n
      --log-file=stdout
      --no-cli

networks:
  matrix:
    driver: bridge
```

```bash
docker compose up -d
# check logs:
docker ps  # wait until matrix-synapse is healthy
docker logs ...
docker compose restart
```

### Create user accounts

Create an admin:
```bash
docker exec -it matrix-synapse register_new_matrix_user \
  --user sample_admin_name_1 \
  --password someReallyLongAndDefinitelyStrongPassword \
  --admin \
  --config /data/homeserver.yaml \
  http://localhost:8008
```
or a regular user (answer 'no' when asked if should be made an admin):
```bash
docker exec -it matrix-synapse register_new_matrix_user \
  --user sample_regular_name_1 \
  --password someReallyLongAndDefinitelyStrongPassword \
  --config /data/homeserver.yaml \
  http://localhost:8008
```

Alternative - read username and password from STDIN (script will ask you) instead of writing the in terminal (will be saved in shell history):
```bash
docker exec -it matrix-synapse register_new_matrix_user \
  --config /data/homeserver.yaml \
  http://localhost:8008
```

Manual: https://manpages.debian.org/testing/matrix-synapse/register_new_matrix_user.1.en.html


### Firewall ufw config

```bash
# check status and disable before changes
sudo ufw status
sudo ufw disable

# set defaults first
sudo ufw default deny incoming
sudo ufw default allow outgoing

# allow required ports
sudo ufw allow 22/tcp    # SSH
sudo ufw allow 80/tcp    # HTTP
sudo ufw allow 443/tcp   # HTTPS
sudo ufw allow 3478/tcp  # calls signalling
sudo ufw allow 3478/udp  # calls signalling
sudo ufw allow 5349/tcp  # calls TLS
sudo ufw allow 49152:49200/udp # calls Relay range

# enable firewall
sudo ufw enable
```

### Install user client

Credentials:
- server: YOUR_DOMAIN,
- username + password, as set in terminal.

After logging in, go to Settings and:
- change your password,
- check Sessions and remove all sessions apart from the current one,
- check Security and Privacy, and remove default Matrix identity server if set(https://vector.im).

Optionally, go to Encryption, turn on Key storage and generate a recovery key. It can be used to restore and decrypt all chats, even in case of losing access to all of your devices. The recovery key should be stored safely. Without it, you will have to reset your keys, so chats will still work, but you won't be able to read past messages. OPTIONAL

#### Windows

Official Element client: https://element.io/en/download

Tested version: 1.12.17, in May 2026.

#### Android

Use the official old Element client `Element Classic`: https://play.google.com/store/apps/details?id=im.vector.app

Tested version: 1.6.54, in May 2026.

## Notes

Random notes:
- deleting files/folders from `~/matrix/synapse/data/` deletes uploaded files, so clients will fail to download it, but everything will still work. Useful for freeing up space,
- after creation, around 672MB RAM is used overall by the whole system, and 1 vCPU spikes at most to 15% during photo upload.

---

### Sources:
- https://stateofsurveillance.org/guides/advanced/matrix-element-self-hosting-guide/
- ChatGPT
