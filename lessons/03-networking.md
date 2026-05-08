---
title: "Docker Networking"
---

# Docker Networking

## Network Types

Docker provides several network drivers:

- **bridge**: Default network for standalone containers
- **host**: Removes network isolation between container and host
- **overlay**: Connects containers across multiple Docker hosts
- **none**: Disables all networking

## Managing Networks

```bash
docker network ls
docker network create my-network
docker network rm my-network
```

## Connecting Containers

### Create a network

```bash
docker network create app-network
```

### Run containers on the network

```bash
docker run -d --name db --network app-network postgres:16-alpine
docker run -d --name api --network app-network my-api:latest
```

### Inspect network

```bash
docker network inspect app-network
```

## Port Mapping

Map container ports to host ports:

```bash
docker run -p 8080:80 nginx
docker run -p 5432:5432 postgres:16-alpine
```

## DNS Resolution

Containers on the same network can reach each other by name:

```bash
# From api container:
ping db
psql -h db -U postgres
```