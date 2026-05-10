---
title: "Docker Networking"
---

# <span class="rainbow">Docker Networking 🌐</span>

## Network Types

Docker provides several network drivers:

<div class="pulse-glow" style="padding: 0.5rem 1rem; background: #f0fdf4; border-radius: 0.5rem; border-left: 4px solid #22c55e;">

- **bridge**: Default network for standalone containers
- **host**: Removes network isolation between container and host
- **overlay**: Connects containers across multiple Docker hosts
- **none**: Disables all networking

</div>

## Managing Networks

```bash
docker network ls
docker network create my-network
docker network rm my-network
```

---

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

---

## <span class="spotlight">Port Mapping</span>

Map container ports to host ports:

```bash
docker run -p 8080:80 nginx
docker run -p 5432:5432 postgres:16-alpine
```

> <span class="blink">⚠️ Warning:</span> Avoid port conflicts by checking which ports are already in use!

---

## DNS Resolution

Containers on the same network can reach each other by name:

```bash
# From api container:
ping db
psql -h db -U postgres
```

<span class="rainbow">🔗 Networking made simple!</span>
