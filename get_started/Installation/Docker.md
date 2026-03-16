# Install with Docker

[To Home](../../README.md)

## Installation of n8n with Docker on Linux

This Markdown section provides a complete setup guide for installing n8n on a Linux system using Docker and Docker Compose. It covers system preparation, Docker installation, project structure, configuration, and running n8n.

## System Requirements

- Linux distribution such as Ubuntu, Debian, CentOS, or Fedora
- Docker installed
- Docker Compose installed
- A dedicated directory for n8n (for example /opt/n8n or ~/n8n)

## Install Docker on Linux

### Update your system

```bash
sudo apt update && sudo apt upgrade -y
```

### Install Docker

```bash
curl -fsSL https://get.docker.com | sudo bash
```

### Enable and start Docker

```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### Add your user to the Docker group

```bash
sudo usermod -aG docker $USER
```

Log out and back in to apply the group change.

## Install Docker Compose

```bash
sudo curl -L "https://github.com/docker/compose/releases/latest/download/docker-compose-$(uname -s)-$(uname -m)" -o /usr/local/bin/docker-compose
sudo chmod +x /usr/local/bin/docker-compose
```

Check the version:

```bash
docker-compose --version
```

## Recommended Project Structure

```Code
n8n/
├─ docker-compose.yml
└─ .env
```

## docker-compose.yml Configuration

```yaml
version: '3.8'

services:
  n8n:
    image: n8nio/n8n:latest
    container_name: n8n
    ports:
      - "5678:5678"
    environment:
      - N8N_BASIC_AUTH_ACTIVE=true
      - N8N_BASIC_AUTH_USER=${N8N_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_PASSWORD}
      - N8N_HOST=${N8N_HOST}
      - N8N_PORT=5678
      - N8N_PROTOCOL=http
      - NODE_ENV=production
    volumes:
      - ./data:/home/node/.n8n
    restart: unless-stopped
```

## .env Configuration

```env
N8N_USER=admin
N8N_PASSWORD=supersecurepassword
N8N_HOST=localhost
```

For production setups with a domain, change N8N_PROTOCOL to https.

## Start n8n

Run n8n in detached mode:

```bash
docker-compose up -d
```

View logs:

```bash
docker-compose logs -f
```

## Access n8n

Local machine:

```Code
http://localhost:5678
```

Remote Linux server:

```Code
http://YOUR-SERVER-IP:5678
```

## Persistent Data

All workflow data is stored in the ./data directory.
This directory should be backed up regularly, especially in production environments.

## Update n8n

```bash
docker-compose pull
docker-compose up -d
```

## Optional Enhancements

- Reverse proxy with SSL (Traefik, Nginx, Caddy)
- PostgreSQL database for improved performance
- Watchtower for automated container updates
- Systemd integration for server-level management
