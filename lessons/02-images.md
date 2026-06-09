---
title: "Docker Images and Containers"
---

# |||rainbow Docker Images & Containers 🐳|||

## |||spotlight Understanding Images|||

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

> |||blink 💡 Tip:||| Use `docker images -a` to see all intermediate layers too!

---

## |||spotlight Working with Containers|||

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

## |||rainbow Best Practices|||

|||callout-yellow
1. Use multi-stage builds for smaller images
2. Don't install unnecessary packages
3. Use specific version tags, not `latest`
4. Leverage layer caching
|||

|||spotlight 🎯 Great job! Keep going!|||
