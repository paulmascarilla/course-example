---
title: "Docker Images and Containers"
---

# <span class="rainbow">Docker Images & Containers 🐳</span>

## <span class="spotlight">Understanding Images</span>

A Docker image is a read-only template with instructions for creating a container. Images are the building blocks of Docker.

### Pulling Images

```bash
docker pull nginx:latest
docker pull postgres:16-alpine
```

### Listing Images

```bash
docker images
docker images -a
```

> <span class="blink">💡 Tip:</span> Use `docker images -a` to see all intermediate layers too!

---

## <span class="spotlight">Working with Containers</span>

### Creating Containers

```bash
docker run nginx
docker run -d --name my-nginx nginx
docker run -p 8080:80 nginx
```

### Managing Containers

```bash
docker ps
docker ps -a
docker stop my-nginx
docker start my-nginx
docker rm my-nginx
```

---

## Building Custom Images

### Dockerfile Example

```dockerfile
FROM node:18-alpine
WORKDIR /app
COPY package*.json ./
RUN npm install
COPY . .
CMD ["npm", "start"]
```

### Building

```bash
docker build -t my-app:latest .
```

---

## <span class="rainbow">Best Practices</span>

<div class="pulse-glow" style="padding: 0.5rem 1rem; background: #fefce8; border-radius: 0.5rem; border-left: 4px solid #eab308;">

1. Use multi-stage builds for smaller images
2. Don't install unnecessary packages
3. Use specific version tags, not `latest`
4. Leverage layer caching

</div>

<span class="spotlight">🎯 Great job! Keep going!</span>
