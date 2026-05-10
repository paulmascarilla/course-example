---
title: "Docker Compose"
---

# <span class="rainbow">Docker Compose 🚀</span>

<span class="spotlight">Docker Compose is a tool for defining and running multi-container applications.</span>

---

## <span class="rainbow">docker-compose.yml</span>

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

## <span class="spotlight">Common Commands</span>

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

> <span class="blink">💡 Pro tip:</span> Combined with a reverse proxy like Nginx, you can load-balance across scaled services!

---

## Environment Variables

Create a `.env` file:

```
POSTGRES_PASSWORD=secretpassword
POSTGRES_DB=mydb
```

---

## <span class="rainbow">Summary</span>

<div class="pulse-glow" style="padding: 0.5rem 1rem; background: #f5f3ff; border-radius: 0.5rem; border-left: 4px solid #8b5cf6;">

Docker Compose simplifies:
- Multi-container applications
- Environment configuration
- Service orchestration
- Local development setups

</div>

<span class="spotlight">🎉 You completed the course! Amazing work!</span>

<div data-explode style="height: 1px;"></div>

<span class="rainbow">✨ Félicitations ! 🎉</span>
