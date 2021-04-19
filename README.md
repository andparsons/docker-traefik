# Secure local docker with Traefik 

Setup trusted certificates for a local dev environment managed by Traefik with DNS resolution on all *.9am.test domains.

## 0. Prerequisites

- [Docker](https://docs.docker.com/docker-for-mac/install/)
- [Homebrew](https://brew.sh/)

## 1. Setup resolvers

```sh
# Setup MacOS to take into account our local docker resolver
sudo mkdir -p /etc/resolver
echo "nameserver 127.0.0.1" | sudo tee -a /etc/resolver/test > /dev/null
```

## 2. Setup a local Root CA

```sh
brew install mkcert
brew install nss # for Firefox

# Setup the local Root CA
mkcert -install
```

## 3. Setup a Traefik container w/ https

```sh
# Create an external network docker, all future containers
# which need to be exposed by domain name should use this network
docker network create external

# Create a local TLS certificate
# *.9am.test will create a wildcard certificate.
# Unfortunately you cannot create *.test wildcard certificate.
mkcert -cert-file acme/local.crt -key-file acme/local.key "9am.test" "*.9am.test"

# Start Traefik
docker-compose up -d

# Go on https://9am.test you should have the traefik web dashboard serve over https
```

## 4. Setup your dev containers

```yaml
# Example craft cms docker file

version: "3.5"

services:
  nginx:
    container_name: "${COMPOSE_PROJECT_NAME:-app}_nginx"
    restart: "unless-stopped"
    image: "nginx:1.19-alpine"
    volumes:
      - "/etc/localtime:/etc/localtime:ro"
      - "/var/run/docker.sock:/tmp/docker.sock:ro"
      - "./docker/nginx/conf.d/default.conf:/etc/nginx/conf.d/default.conf:ro"
      - "./web:/var/www/html/web:rw"
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-app}.tls=true"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-app}.entrypoints=https"
      - "traefik.http.routers.${COMPOSE_PROJECT_NAME:-app}.rule=Host(`${SITE:-app.9am.test}`)"
    depends_on:
      - php
    networks:
      - external
      - internal

  php:
    container_name: "${COMPOSE_PROJECT_NAME:-app}_php"
    restart: "unless-stopped"
    image: "09am/craft-php:7.4-fpm-alpine"
    volumes:
      - "/etc/localtime:/etc/localtime:ro"
      - "./:/var/www/html:rw"
    depends_on:
      - mysql
    networks:
      - external # Needed for external networking like curl in composer
      - internal

  mysql:
    container_name: "${COMPOSE_PROJECT_NAME:-app}_db"
    restart: "unless-stopped"
    image: "mariadb:10.1"
    environment:
      MYSQL_ROOT_PASSWORD: "root"
      MYSQL_DATABASE: "${DB_DATABASE:-db_name}"
      MYSQL_USER: "${DB_USER:-db_user}"
      MYSQL_PASSWORD: "${DB_PASSWORD:-db_pass}"
    ports:
      - "3306:3306"
    expose:
      - "3306"
    volumes:
      - "/etc/localtime:/etc/localtime:ro"
      - "db-data:/var/lib/mysql:rw"
      - "./docker/db/seed/:/docker-entrypoint-initdb.d:rw"
    networks:
      - internal

volumes:
  db-data:

networks:
  internal:
    internal: true
  external:
    external: true
```
```dotenv
# The environment Craft is currently running in ('dev', 'staging', 'production', etc.)
ENVIRONMENT="dev"
# The environment for webpack
NODE_ENV="development"

# The URI segment that tells Craft to load the control panel
CP_TRIGGER="admin"

# The application ID used to to uniquely store session and cache data, mutex locks, and more
APP_ID="CraftCMS--123"

# The secure key Craft will use for hashing and encrypting data
SECURITY_KEY="A-Z"

# The database server name or IP address (usually this is 'localhost' or '127.0.0.1')
DB_SERVER="mysql"

# The database driver that will be used ('mysql' or 'pgsql')
DB_DRIVER="mysql"

# The port to connect to the database with. Will default to 5432 for PostgreSQL and 3306 for MySQL.
DB_PORT="3306"

# The database schema that will be used (PostgreSQL only)
DB_SCHEMA="public"

# The name of the database to select
DB_DATABASE="app_dev"

# The database username to connect with
DB_USER="app_user"

# The database password to connect with
DB_PASSWORD="app_pass"

# The prefix that should be added to generated table names (only necessary if multiple things are sharing the same database)
DB_TABLE_PREFIX=""

# The unique short name used by docker 
COMPOSE_PROJECT_NAME="app"

# The local host without a protocol, used by traefik
SITE="app.9am.test"

# The local host with a secure protocol, used by craft
PRIMARY_SITE_URL="https://app.9am.test"
```
