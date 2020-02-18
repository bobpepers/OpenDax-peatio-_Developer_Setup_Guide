# OpenDax-peatio-_Developer_Setup_Guide

# Ubuntu 18.04
# Peatio 2.3-stable
## Assuming you have Docker, Docker-Compose, RVM installed, NVM

# NVM, NODEJS & YARN
curl -o- https://raw.githubusercontent.com/creationix/nvm/v0.33.0/install.sh | bash

export NVM_DIR="/home/app/.nvm"
[ -s "$NVM_DIR/nvm.sh" ] && \. "$NVM_DIR/nvm.sh"  # This loads nvm

nvm i 12
nvm use 12

npm i -g yarn 

# RVM

## Source RVM
source /home/app/.rvm/scripts/rvm

#Docker

## Install Docker
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu bionic stable"
sudo apt update
apt-cache policy docker-ce
sudo apt install docker-ce
sudo usermod -aG docker $USER
sudo usermod -a -G docker app
sudo systemctl enable docker
sudo systemctl start docker

### Restart terminal to make docker install active

## Docker Commands

### Docker remove Volumes
docker volume prune

### Docker list volumes
docker volume ls

### Docker list containers
docker ps

### Stop all Docker Containers
docker stop $(docker ps -a -q)

### Remove All docker contaienrs
docker rm $(docker ps -a -q)

# Docker-Compose

## Install Docker-Compose
sudo curl -L https://github.com/docker/compose/releases/download/1.21.2/docker-compose-`uname -s`-`uname -m` -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
docker-compose --version

# Add sudo user app

## Add User
sudo adduser app

## Give user app sufficient privledges
sudo usermod -a -G sudo app

## Login to app user
su app

## Give permanent Root Privledges (optional - Skip this if your RVM does not complain about privledges)
sudo -s

# Add the local domain to /etc/hosts
echo "0.0.0.0 app.local" | sudo tee -a /etc/hosts

# Clone the repositories

git clone https://github.com/openware/peatio
git checkout 2-3-stable
git clone https://github.com/openware/barong
git clone https://github.com/rubykube/applogic
git clone https://github.com/openware/baseapp

# GeoLite

## Download

https://download.maxmind.com/app/geoip_download?edition_id=GeoLite2-Country&suffix=tar.gz&license_key=T6ElPBlyOOuCyjzw

## export GeoLite2-Country.mmdb

mkdir /home/app/geolite
- copy GeoLite2-Country.mmdb in /home/app/geolite

# LOCAL DEVELOPMENT

## Install Peatio for local development

### Give permanent Root Privledges (optional - Skip this if your RVM does not complain about privledges)
sudo -s

### Install Ruby 2.6.5
rvm install 2.6.5

### Use Ruby 2.6.5
rvm use 2.6.5

### Instal Mysql Lib
sudo apt-get install libmysqlclient-dev

### Navigate to peatio folder
cd peatio

### Bundle install
bundle install

### Create keys with openssl (Barong key)
openssl genrsa -out config/secrets/private 2048
openssl rsa -in config/secrets/private -outform PEM -pubout -out config/secrets/public

### Create key with peatio script (this is the same as creating 2048 rsa key) (AppLogic key)
bundle exec peatio security keygen --path=config/secrets


### Unset DOCKER_HOST_IP
unset DOCKER_HOST_IP

### Create env var to connect docker container to host
export DOCKER_HOST_IP=$(ip route | grep docker0 | awk '{print $9}')

### Echo DOCKER_HOST_IP to see if it properly set
echo $DOCKER_HOST_IP

### Edit config/backend.yml (edit gateway service with new config)
nano config/backend.yml

```
  
  gateway:
    restart: always
    image: envoyproxy/envoy:v1.10.0
    network_mode: "host"
    extra_hosts:
      - "host.docker.internal:$DOCKER_HOST_IP"
    volumes:
      - /home/app/peatio/config/gateway:/etc/envoy/
    command: /usr/local/bin/envoy -l info -c /etc/envoy/envoy.yaml
    labels:
      traefik.enable: true
      traefik.frontend.rule: "PathPrefix:/api/, /admin, /assets/;Host:app.local"
      traefik.default.port: 9002
      traefik.default.protocol: http
      traefik.wss.frontend.rule: "Host:app.local"
      traefik.wss.protocol: ws
      traefik.wss.port: 9002

```

### Edit config/gateway/envoy.yaml
Remove All files in this folder and replace with new envoy.yaml file

envoy.yaml

```

static_resources:
  listeners:
  - address:
      socket_address:
        address: 0.0.0.0
        port_value: 9002
    filter_chains:
    - filters:
      - name: envoy.http_connection_manager
        config:
          codec_type: auto
          stat_prefix: ingress_http
          route_config:
            name: local_route
            virtual_hosts:
            - name: backend
              domains:
              - "*"
              cors:
                allow_origin:
                - "*"
                allow_methods: "PUT, GET, POST"
                allow_headers: "content-type, x-grpc-web"
                filter_enabled:
                  default_value:
                    numerator: 100
                    denominator: HUNDRED
                  runtime_key: cors.www.enabled
              routes:
              - match:
                  prefix: "/api/v2/barong"
                route:
                  cluster: barong
                  prefix_rewrite: "/api/v2/"
              - match:
                  prefix: "/api/v2/applogic"
                route:
                  cluster: applogic
                  prefix_rewrite: "/api/v2/"
              - match:
                  prefix: "/api/v2/arke"
                route:
                  cluster: arke
                  prefix_rewrite: "/api/v2/"
              - match:
                  prefix: "/api/v2/peatio"
                route:
                  cluster: peatio
                  prefix_rewrite: "/api/v2/"
              - match:
                  prefix: "/admin"
                route:
                  cluster: peatio
              - match:
                  prefix: "/assets/"
                route:
                  cluster: peatio
              - match:
                  prefix: "/api/v2/ranger/public"
                route:
                  cluster: ranger
                  prefix_rewrite: "/"
                  upgrade_configs:
                    upgrade_type: "websocket"
              - match:
                  prefix: "/api/v2/ranger/private"
                route:
                  cluster: ranger
                  prefix_rewrite: "/"
                  upgrade_configs:
                    upgrade_type: "websocket"
          http_filters:
          - name: envoy.cors
            typed_config: {}
          - name: envoy.ext_authz
            config:
              failure_mode_allow: true
              with_request_body:
                max_request_bytes: 90000000
                allow_partial_message: true
              http_service:
                authorization_request:
                  allowed_headers:
                    patterns:
                    - exact: cookie
                    - exact: x-auth-apikey
                    - exact: x-auth-nonce
                    - exact: x-auth-signature
                    - exact: user-agent
                    - exact: x-forwarded-host
                    - exact: x-forwarded-for
                    - exact: from
                    - exact: x-forwarded-proto
                    - exact: proxy-authorization
                authorization_response:
                  allowed_upstream_headers:
                    patterns:
                    - exact: authorization
                  allowed_client_headers:
                    patterns:
                    - exact: set-cookie
                    - exact: proxy-authenticate
                    - exact: www-authenticate
                    - exact: location
                path_prefix: "/api/v2/auth"
                server_uri:
                  cluster: barong
                  timeout: 1.000s
                  uri: http://host.docker.internal:9001
          - name: envoy.router
            config: {}
    perConnectionBufferLimitBytes: 10000000
  clusters:
  - name: barong
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: host.docker.internal
        port_value: 9001
  - name: applogic
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: host.docker.internal
        port_value: 8999
  - name: arke
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: arke
        port_value: 8081
  - name: peatio
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: host.docker.internal
        port_value: 9000
  - name: ranger
    connect_timeout: 0.25s
    type: strict_dns
    lb_policy: round_robin
    hosts:
    - socket_address:
        address: ranger
        port_value: 8080
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 9099


```

### Edit config/templates/management_api_v1.yml.erb

```


keychain:
  applogic:
    algorithm: RS256
    value: <%= ENV['APPLOGIC_PUBLIC_KEY'] %>

scopes:
  read_operations:
    mandatory_signers:
      - applogic
    permitted_signers:
      - applogic

jwt: {}


```

### Unset peatio env vars
unset APPLOGIC_PUBLIC_KEY
unset EVENT_API_JWT_PRIVATE_KEY
unset EVENT_API_JWT_PUBLIC_KEY
unset JWT_PUBLIC_KEY
unset BARONG_DOMAIN
unset ADMIN
unset URL_HOST
unset MANAGEMENT_API_V1_CONFIG
unset BARONG_MAXMINDDB_PATH

### Set peatio key env vars
export APPLOGIC_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/rsa-key.pub| base64 -w0)
export EVENT_API_JWT_PRIVATE_KEY=$(cat /home/app/peatio/config/secrets/private| base64 -w0)
export EVENT_API_JWT_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/public| base64 -w0)
export JWT_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/public| base64 -w0)
export BARONG_DOMAIN=173.249.7.60:9001
export ADMIN=test@runebase.io
export URL_HOST=app.local
export MANAGEMENT_API_V1_CONFIG=/home/app/peatio/config/management_api_v1.yml
export JWT_AUDIENCE=peatio,barong
export BARONG_MAXMINDDB_PATH=/home/app/geolite/GeoLite2-Country.mmdb

### Setup docker containers for services
./bin/setup

### Stop all daemons
god stop

### Start all daemons
god -c lib/daemons/daemons.god

### start peatio server
rails s -b 0.0.0.0 -p 9000


## Install Barong for local development

### Give permanent Root Privledges (optional - Skip this if your RVM does not complain about privledges)
sudo -s

### Use Ruby 2.6.5
rvm use 2.6.5

### Update Bundler
bundle update --bundler

### Navigate to Barong directory
cd /home/app/barong

### Bundle install
bundle install

### Create config/templates/management_api.yml.erb

```


keychain:
  applogic:
    algorithm: RS256
    value: <%= ENV['APPLOGIC_PUBLIC_KEY'] %>
    
scopes:
  read_users:
    mandatory_signers:
      - applogic
    permitted_signers:
      - applogic


```

### unset barong env vars
unset APPLOGIC_PUBLIC_KEY
unset JWT_PRIVATE_KEY_PATH
unset JWT_PUBLIC_KEY
unset EVENT_API_JWT_PRIVATE_KEY
unset ADMIN_EMAIL
unset ADMIN_PASSWORD
unset APP_NAME

### set barong private key path
export APPLOGIC_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/rsa-key.pub| base64 -w0)
export JWT_PRIVATE_KEY_PATH="/home/app/peatio/config/secrets/private"
export JWT_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/public| base64 -w0)
export EVENT_API_JWT_PRIVATE_KEY=$(cat /home/app/peatio/config/secrets/private| base64 -w0)
export EVENT_API_JWT_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/public| base64 -w0)
export APP_NAME=App Local

### Export Login details for barong
export ADMIN_EMAIL=test@runebase.io
export ADMIN_PASSWORD=Qwerty123\@321\!321

### Initialize Config
bin/init_config

### Create Barong DB
bundle exec rake db:create db:migrate

### Seed DB with login details
rake db:seed

### Install Modules
yarn install

### start barong server
rails s -b 0.0.0.0 -p 9001


## Install AppLogic for local development

### Give permanent Root Privledges (optional - Skip this if your RVM does not complain about privledges)
sudo -s

### Install Ruby 2.5.0
rvm install "ruby-2.5.0"

### Use Ruby 2.5.0
rvm use 2.5.0

### Navigate to Barong directory
cd /home/app/applogic

### unset AppLogic env vars
unset BARONG_URL
unset PEATIO_URL
unset REDIS_URL
unset PORT
unset EVENT_API_JWT_PUBLIC_KEY
unset JWT_PUBLIC_KEY
unset JWT_PRIVATE_KEY

unset EVENT_API_APPLICATION
unset EVENT_API_EVENT_CATEGORY
unset EVENT_API_EVENT_NAME

### Export AppLogic env vars
export BARONG_URL=http://app.local:9001
export PEATIO_URL=http://app.local:9000
#### export REDIS_URL=
export PORT=8999
export EVENT_API_JWT_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/public| base64 -w0)
export JWT_PUBLIC_KEY=$(cat /home/app/peatio/config/secrets/public| base64 -w0)
export JWT_PRIVATE_KEY=$(cat /home/app/peatio/config/secrets/rsa-key| base64 -w0)

export EVENT_API_APPLICATION=barong
export EVENT_API_EVENT_CATEGORY=model
export EVENT_API_EVENT_NAME=profile.created

### Init Config
./bin/init_config

### Bundle install
bundle install

### Create and Seed DB
rake db:create db:migrate db:seed

### Make Logs Writable if needed
sudo chmod 0664 /home/app/applogic/log/development.log

### Start Server
rails s -b 0.0.0.0 -p 8999

## Install BaseApp for local development

### Use NODE 12
nvm use 12

### Install Dependencies
yarn install

### Run tests
yarn test

### Start server
yarn start

### Your Barong Login Details
username: test@runebase.io
password: Qwerty123@321!321