## Docker standalone traefik reverse proxy

contains sample traefik reverse proxy `Note this works for docker standalone using docker compose`

## Use on swarm mode

To use on docker swarm, some changes need to be made.
Traefik needs extra configs example.

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:latest
    restart: unless-stopped
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--api"

      # Extra config enable swarm mode and network
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.network=traefik_default"
      # -------------------------------------------- end swarm config

    ports:
      - "80:80"
      - "443:443"
      - "8080:8080" # Only needed if you want dashboard for traefik, used together with --api.insecure=true
    
    # Add default network, will be used by any container to be exposed
    networks:
      - default
    # -------------------------------------------- end networks
    
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    
    # How to deploy to portainer
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: ["node.role == manager"]
    # -------------------------------------------- end deploy

# Network
networks:
  default:
# -------------------------------------------- end networks
```

For each of the apps to be expose, labels needs to be inside the deploy section. Also we need to add traefik network to the container. Example below

```yaml
version: "3.9"

services:
  nginx:
    image: nginx
    hostname: "{{.Node.Hostname}}_nginx"
    environment:
      - NODE_HOSTNAME={{.Node.Hostname}}
    configs:
      - source: nginx-config.sh
        target: /docker-entrypoint.d/setup.sh
        mode: 0555
    deploy:
      mode: global # or use replicated with number of replicas
      placement:
        constraints: ["node.role == worker"]
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.rule=Host(`nginx.localhost`)"
        - "traefik.http.routers.nginx.entrypoints=web"
        - "traefik.http.services.nginx.loadbalancer.server.port=80"
        - "traefik.http.routers.nginx.service=nginx"

configs:
  nginx-config.sh:
    external: true

networks:
  default:
    name: traefik_default
    external: true

```


This is sample config for nginx, allow to set the hostname in default page so that can see load balance in action

```bash
#!/bin/bash
set -x
sed -i "s/Welcome to nginx\!/Welcome from container $(hostname)/g" /usr/share/nginx/html/index.html
```


## Lets encrypt SSL/TLS

Traefik needs more config to enable SSL, example below sections highlighted

```yaml
version: "3.9"

services:
  traefik:
    image: traefik:latest
    restart: unless-stopped
    command:
      - "--log.level=DEBUG"
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--api"

      # Extra config enable swarm mode and network
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.network=traefik_default"
      # -------------------------------------------- end swarm config

      # Enable ACME (Let's Encrypt): automatic SSL.
      - "--certificatesresolvers.letsencrypt.acme.email=your.name@domain.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge.entrypoint=web"
      # Global redirect to https
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"

    ports:
      - "80:80"
      - "443:443"
      - "8080:8080" # Only needed if you want dashboard for traefik, used together with --api.insecure=true
    
    # Add default network, will be used by any container to be exposed
    networks:
      - default
    # -------------------------------------------- end networks
    
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"

      # Mount SSL volume
      - letsencrypt:/letsencrypt
      # ------------------ end mount
    
    # How to deploy to portainer
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: ["node.role == manager"]

      # Under deploy section, enable http challenge to get ssl certificate
      labels:
        - "traefik.enable=true"
        - "traefik.http.routers.dashboard-http.rule=Host(`domain.com`)"
        - "traefik.http.routers.dashboard-http.entrypoints=web"
        - "traefik.http.routers.dashboard-http.service=api@internal"
      # -------------------------------------------- end labels

    # -------------------------------------------- end deploy

# Volumes SSL
volumes:
  letsencrypt: {}
# -------------------------------------------- end volumes

# Network
networks:
  default:
# -------------------------------------------- end networks

```

For other services we need to change entrypoint from web to websecure and also add below line

```yaml
        - "traefik.http.routers.service_name.tls.certresolver=letsencrypt"
```

Example below

```yaml
        - "traefik.enable=true"
        - "traefik.http.routers.nginx.rule=Host(`nginx.domain.com`)"
        - "traefik.http.routers.nginx.entrypoints=websecure"
        - "traefik.http.services.nginx.loadbalancer.server.port=80"
        - "traefik.http.routers.nginx.service=nginx"
        - "traefik.http.routers.nginx.tls.certresolver=letsencrypt"

```
