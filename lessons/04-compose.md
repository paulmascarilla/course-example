---
title: "Docker Compose"
---

# Docker Compose

Docker Compose is a tool for defining and running multi-container applications.

---

## docker-compose.yml

```yaml
version: '3.8'

services:
  web:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://postgres:password@db:5432/mydb
    depends_on:
      - db

  db:
    image: postgres:16-alpine
    environment:
      - POSTGRES_PASSWORD=password
      - POSTGRES_DB=mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data

volumes:
  postgres-data:
```

---

## Common Commands

```bash
# Start services
docker-compose up
docker-compose up -d

# Stop services
docker-compose down
docker-compose down -v

# View logs
docker-compose logs -f

# Rebuild
docker-compose up --build
```

---

## Scaling

```bash
docker-compose up --scale web=3
```

> **Pro tip:** Combined with a reverse proxy like Nginx, you can load-balance across scaled services!

---

## Environment Variables

Create a `.env` file:

```
POSTGRES_PASSWORD=secretpassword
POSTGRES_DB=mydb
```

---

## Summary

Docker Compose simplifies:
- Multi-container applications
- Environment configuration
- Service orchestration
- Local development setups
