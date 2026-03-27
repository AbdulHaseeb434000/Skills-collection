# Docker CLI Reference

## docker build

```bash
docker build [OPTIONS] PATH | URL

# Common options
-t, --tag      name:tag               # Tag the image
-f, --file     Dockerfile.prod        # Specify Dockerfile path
--no-cache                            # Ignore all cached layers
--build-arg    KEY=VALUE              # Set ARG values
--target       stage-name             # Build up to a specific stage (multi-stage)
--platform     linux/amd64,linux/arm64 # Build for specific platform(s)
--pull                                # Always pull base images
--progress     plain                  # Verbose output (plain | tty | auto)

# Examples
docker build -t myapp:1.0.0 .
docker build -t myapp:dev -f Dockerfile.dev --build-arg DEBUG=true .
docker build --target builder -t myapp:builder .
DOCKER_BUILDKIT=1 docker build --secret id=token,src=./token.txt .
```

## docker buildx (extended build, BuildKit)

```bash
# Create and use a builder instance
docker buildx create --use --name mybuilder

# Multi-platform build
docker buildx build --platform linux/amd64,linux/arm64 -t myapp:latest --push .

# Build with GitHub Actions cache
docker buildx build --cache-from type=gha --cache-to type=gha,mode=max -t myapp .

# Inspect builder capabilities
docker buildx inspect --bootstrap
```

## docker run

```bash
docker run [OPTIONS] IMAGE [COMMAND]

# Common options
-d, --detach                          # Run in background
-it                                   # Interactive + tty (for shells)
--rm                                  # Auto-remove on exit
--name    mycontainer                 # Assign name
-p, --publish  8080:80               # Publish port (host:container)
-e, --env   KEY=VALUE                # Set env var
--env-file  .env                     # Load env from file
-v, --volume  mydata:/data           # Mount volume
--mount   type=bind,src=./,dst=/app  # Explicit mount syntax
--network  mynetwork                  # Connect to network
--user    1001:1001                   # Run as user:group
--read-only                          # Read-only root filesystem
--cap-drop  ALL                      # Drop capabilities
--cap-add   NET_BIND_SERVICE         # Add specific capability
--memory   512m                      # Memory limit
--cpus     1.5                       # CPU limit
--restart  unless-stopped            # Restart policy
--health-cmd "curl -f http://localhost/health" # Override healthcheck
--no-healthcheck                     # Disable healthcheck

# Examples
docker run -d --name myapp -p 3000:3000 --env-file .env myapp:latest
docker run --rm -it --entrypoint sh myapp:latest   # Debug: get a shell
docker run --rm -v $(pwd):/work -w /work node:20 npm test  # One-off task
```

## docker exec

```bash
docker exec [OPTIONS] CONTAINER COMMAND

# Get a shell in running container
docker exec -it myapp sh      # Alpine/slim
docker exec -it myapp bash    # Debian/Ubuntu

# Run a command
docker exec myapp env         # Print env vars
docker exec myapp cat /etc/hosts
docker exec -u root myapp sh  # Execute as root (even if container runs as user)
```

## docker logs

```bash
docker logs [OPTIONS] CONTAINER

-f, --follow          # Stream live logs
--tail  100           # Last N lines
--since 2024-01-01    # Since timestamp
--since 1h            # Since duration
-t, --timestamps      # Show timestamps

docker logs -f --tail=200 myapp
docker logs --since=30m myapp 2>&1 | grep ERROR
```

## docker inspect

```bash
docker inspect CONTAINER_OR_IMAGE

# Extract specific fields (uses Go template)
docker inspect -f '{{.State.Status}}' myapp
docker inspect -f '{{.NetworkSettings.IPAddress}}' myapp
docker inspect -f '{{json .Config.Env}}' myapp | jq .
docker inspect -f '{{range .Mounts}}{{.Source}} -> {{.Destination}}{{"\n"}}{{end}}' myapp
```

## docker stats

```bash
docker stats [CONTAINER...]

# Live resource usage: CPU, memory, net I/O, block I/O
docker stats                  # All running containers
docker stats myapp            # Specific container
docker stats --no-stream      # Snapshot (not live)
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

## docker cp

```bash
# Copy files between container and host
docker cp myapp:/var/log/app.log ./app.log
docker cp ./config.json myapp:/app/config.json
```

## Image Management

```bash
# List images
docker images
docker image ls --format "table {{.Repository}}\t{{.Tag}}\t{{.Size}}"

# Remove images
docker rmi myapp:old
docker image prune           # Remove dangling (untagged) images
docker image prune -af       # Remove all unused images

# Image details / layer sizes
docker image history myapp:latest
docker image inspect myapp:latest

# Tag and push
docker tag myapp:latest registry.example.com/team/myapp:1.0.0
docker push registry.example.com/team/myapp:1.0.0

# Pull
docker pull nginx:1.25-alpine
docker pull --platform linux/arm64 nginx:1.25-alpine

# Save/load (for airgapped environments)
docker save myapp:latest | gzip > myapp.tar.gz
docker load < myapp.tar.gz

# Scan for vulnerabilities (requires Docker Scout or Snyk)
docker scout cves myapp:latest
```

## Container Management

```bash
docker ps                    # Running containers
docker ps -a                 # All containers (including stopped)
docker ps --format "table {{.Names}}\t{{.Status}}\t{{.Ports}}"

docker start myapp
docker stop myapp            # Sends SIGTERM, then SIGKILL after grace period
docker kill myapp            # Sends SIGKILL immediately
docker restart myapp

docker rm myapp              # Remove stopped container
docker rm -f myapp           # Force remove running container
docker container prune       # Remove all stopped containers
```

## Network Management

```bash
docker network ls
docker network create --driver bridge mynetwork
docker network connect mynetwork myapp
docker network disconnect mynetwork myapp
docker network inspect mynetwork
docker network rm mynetwork
docker network prune         # Remove unused networks
```

## Volume Management

```bash
docker volume ls
docker volume create mydata
docker volume inspect mydata
docker volume rm mydata
docker volume prune          # Remove unused volumes
docker volume prune -af      # Remove all volumes (⚠️ data loss)
```

## System & Cleanup

```bash
docker system df             # Disk usage by images, containers, volumes, cache
docker system df -v          # Verbose breakdown

# Clean up (in order of increasing destructiveness)
docker container prune       # Stopped containers
docker image prune           # Dangling images
docker image prune -a        # All unused images
docker volume prune          # Unused volumes
docker network prune         # Unused networks
docker builder prune         # Build cache
docker system prune          # containers + dangling images + networks + build cache
docker system prune -af --volumes  # ⚠️ EVERYTHING unused
```

---

## Docker Compose CLI

```bash
# Start services (detached)
docker compose up -d

# Start specific service
docker compose up -d app

# Rebuild and start
docker compose up -d --build

# Force recreate (even if config unchanged)
docker compose up -d --force-recreate

# Stop and remove containers (keep volumes)
docker compose down

# Stop and remove containers + volumes (⚠️ data loss)
docker compose down -v

# Stop without removing
docker compose stop

# View logs
docker compose logs -f
docker compose logs -f app db    # Specific services

# Shell into a service
docker compose exec app sh

# Run a one-off command in a service container
docker compose run --rm app python manage.py migrate

# List running services
docker compose ps

# Show service config (merged)
docker compose config

# Pull latest images for all services
docker compose pull

# Scale a service
docker compose up -d --scale worker=3

# Restart a service
docker compose restart app

# Use multiple compose files
docker compose -f docker-compose.yml -f docker-compose.prod.yml up -d
```
