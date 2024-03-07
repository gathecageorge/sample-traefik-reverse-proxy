contains sample traefik reverse proxy

### Note this works for docker standalone using docker compose

### To use on docker swarm, some changes need to be made.

## Difference with standalone: Can deploy this portainer stack using portainer

Traefik needs extra configs example

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
      - "--providers.docker.swarmMode=true"
      - "--providers.docker.network=traefik_default"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"
    networks:
      - default
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
    deploy:
      mode: replicated
      replicas: 1
      placement:
        constraints: ["node.role == manager"]

networks:
  default:
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
      mode: global # Run in all nodes that match constaint
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
