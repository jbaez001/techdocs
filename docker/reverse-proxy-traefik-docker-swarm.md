<!-- omit in toc -->
# Reverse Proxy with Traefik and LetsEncrypt (SSL Certificates)

<!-- omit in toc -->
## Table of Contents

- [Portainer Stacks](#portainer-stacks)
  - [Stack: Backoffice](#stack-backoffice)
  - [Stack: {YOURDOMAIN}](#stack-yourdomain)
- [Installation](#installation)
  - [Step 1: Install Portainer](#step-1-install-portainer)
  - [Step 2: Create the "public" network](#step-2-create-the-%22public%22-network)
  - [Step 3: Deploy the Reverse Proxy](#step-3-deploy-the-reverse-proxy)
  - [Step 4: Deploy the Application](#step-4-deploy-the-application)

## Portainer Stacks

The stacks here may be used with Portainer even without docker-compose being installed. They may also be used with
docker-compose as well. The filenames are provided in the event that it is preferred to save them into a file as opposed
to using Portainer.

### Stack: Backoffice

This stack is pretty much for running the reverse-proxy. Any backoffice related containers/services should be added here as well.
Individual applications/websites/domains would ideally not be contained here.

Traefik will be published with ports 80 and 443.

<!-- omit in toc -->
#### Variable Definition

| Variable | Definition | Note(s) |
| -------- | ---------- | ------- |
| {TLS_CHALLENGE_EMAIL} | E-mail for domains that use the TLS challenge for LetsEncrypt Certificates | Wildcard certificates are not supported. |

<!-- omit in toc -->
#### backoffice.yml

```yaml
version: '2'

services:
  reverse-proxy:
    image: traefik:v2.2
    command: 
      - "--log.level=WARN"
      - "--log.filePath=/var/log/traefik.log"
      - "--accesslog=true"
      - "--accesslog.filepath=/var/log/access.log"
      - "--accesslog.bufferingsize=100"
      # - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--providers.docker.network=public"
      - "--entryPoints.metrics.address=:8082"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.tlsChallenge.acme.tlschallenge=true"
      - "--certificatesresolvers.tlsChallenge.acme.email={TLS_CHALLENGE_EMAIL}"
      - "--certificatesresolvers.tlsChallenge.acme.storage=/letsencrypt/acme.json"
    restart: unless-stopped
    networks:
      - public
    ports:
      - "80:80"
      - "443:443"
      -#  "8082:8082"
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - vol_certificates:/letsencrypt
      - vol_logs:/var/log

volumes:
  vol_certificates:
  vol_logs:

networks:
  public:
    external: 
      name: public
```

### Stack: {YOURDOMAIN}

<!-- omit in toc -->
#### Variable Definition

| Variable | Definition | Note(s) |
| -------- | ---------- | ------- |
| {YOURDOMAIN} | yourdomain.com | Do not add www |
| {SERVICE_NAME} | yourdomain | Traefik requires a unique service/loadbalancer name. This is how it knows to get to the applications. |

<!-- omit in toc -->
#### {YOURDOMAIN}.yml

```yaml
version: '2'

services:
  mariadb:
    image: linuxserver/mariadb
    restart: unless-stopped
    volumes:
      - vol_database:/config
    expose:
      - "3306"
    networks:
      - database
    environment:
      MYSQL_ROOT_PASSWORD: S3cr3tS4ls4
      MYSQL_DATABASE: ghost
      MYSQL_USER: salsa
      MYSQL_PASSWORD: s4ls4
      PUID: 10000
      PGID: 10000

  ghost:
    depends_on:
      - mariadb
    image: ghost
    restart: unless-stopped
    volumes:
      - vol_ghost:/var/lib/ghost/content
    expose:
      - "2368"
    networks:
      - database
      - public
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.{SERVICE_NAME}.loadbalancer.server.port=2368"
      - "traefik.http.routers.{SERVICE_NAME}.rule=HostRegexp(`{YOURDOMAIN}`, `{subdomains:(www)}.{YOURDOMAIN}`)"
      - "traefik.http.routers.{SERVICE_NAME}.entrypoints=websecure"
      - "traefik.http.routers.{SERVICE_NAME}.tls.certresolver=tlsChallenge"
      - "traefik.http.routers.{SERVICE_NAME}.tls.domains[0].main={YOURDOMAIN}"
      - "traefik.http.routers.{SERVICE_NAME}.tls.domains[0].sans=www.{YOURDOMAIN}"
    environment:
      url: {YOURDOMAIN}
      database__client: mysql
      database__connection__host: mariadb
      database__connection__user: salsa
      database__connection__password: s4ls4
      database__connection__database: ghost

volumes:
  vol_database:
  vol_ghost:

networks:
  database:
    attachable: true

  public:
    external:
      name: public
```

## Installation

### Step 1: Install Portainer

The script below will run Portainer with ports 8000 and 9000 published.

<!-- omit in toc -->
#### Bash Script

```bash
docker volume create vol_portainer
docker run -d \
  -p 8000:8000 \
  -p 9000:9000 \
  --name=portainer \
  --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v vol_portainer:/data portainer/portainer
```

Once the Portainer container is up and running, you will want to then navigate over to it using your web browser and the IP of the server/VPS using port 9000. Follow the initial basic user setup then go to "Stacks" for the next steps.

### Step 2: Create the "public" network

The network may be created through Portainer or via cli.

<!-- omit in toc -->
#### Bash Script

```bash
docker network create public
```

### Step 3: Deploy the Reverse Proxy

You will want to create a new stack in Portainer with a unique name of your choosing. 
For this particular example, the name "backoffice" is used.

Next, just copy/paste the contents of "backoffice.yml" (provided earlier above) while making sure to replace the correct variables.

### Step 4: Deploy the Application

For your application, you will want to create another stack in Portainer. The application port will need to be specified and we will also
need to tell traefik how to find this application. This is done via labels as seen in the sample configuration below.

Earlier above a full stack example was given of mariadb and ghost. The configuration below is a template on that one could addapt to most if not all applications.

<!-- omit in toc -->
#### Variable Definition

| Variable | Definition | Note(s) |
| -------- | ---------- | ------- |
| {APPLICATION_PORT} | 80 | Port that the application listens to. |
| {SERVICE_NAME} | yourdomain | Traefik service name |

<!-- omit in toc -->
#### application.yaml

```yaml
version: '2'

services:
  {APPLICATION_NAME}:
    image: {APPLICATION_IMAGE}
    expose:
      - "{APPLICATION_PORT}"
    labels:
      - "traefik.enable=true"
      - "traefik.http.services.{SERVICE_NAME}.loadbalancer.server.port={APPLICATION_PORT}"
      - "traefik.http.routers.{SERVICE_NAME}.rule=HostRegexp(`{YOURDOMAIN}`, `{subdomains:(www)}.{YOURDOMAIN}`)"
      - "traefik.http.routers.{SERVICE_NAME}.entrypoints=websecure"
      - "traefik.http.routers.{SERVICE_NAME}.tls.certresolver=tlsChallenge"
      - "traefik.http.routers.{SERVICE_NAME}.tls.domains[0].main={YOURDOMAIN}"
      - "traefik.http.routers.{SERVICE_NAME}.tls.domains[0].sans=www.{YOURDOMAIN}"

networks:
  public:
    external:
      name: public
```
