---
title: "Introduction to Docker"
---

# <span class="rainbow">Introduction to Docker 🎉</span>

<span class="spotlight">Docker is a platform for developing, shipping, and running applications inside containers.</span> Containers are lightweight, portable, and self-sufficient units that include everything needed to run a piece of software.

---

## <span class="blink">✨ What is a Container?</span>

A container is a standardized unit of software that packages up code and all its dependencies so the application runs quickly and reliably from one computing environment to another.

> **Fun fact:** Containers have been around since the 1970s (chroot), but Docker made them popular in 2013!

---

## Benefits of Docker

<div class="pulse-glow" style="padding: 1rem; border-radius: 0.5rem; margin: 1rem 0;">

- **Consistency**: Same environment from development to production
- **Isolation**: Applications run in isolated environments
- **Portability**: Run anywhere Docker is installed
- **Efficiency**: Lightweight compared to virtual machines

</div>

## Installing Docker

### Linux (Ubuntu)

```bash
sudo apt update
sudo apt install docker.io
```

### macOS/Windows

Download Docker Desktop from [docker.com](https://www.docker.com/products/docker-desktop)

## Verifying Installation

```bash
docker --version
docker run hello-world
```

---

<div data-explode style="height: 1px;"></div>

<span class="rainbow">🎯 Congratulations on completing the first lesson!</span>
