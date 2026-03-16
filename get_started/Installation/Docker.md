# Install with Docker [Linux]

[To Home](../../README.md)

> **Who is this guide for?**  
> For everyone who wants to run n8n locally or on a server – even without any prior knowledge. Every step is explained in detail.
>
> For Windows/macOS users, only the software "*Docker Desktop*" is needed [docker.com](https://www.docker.com/get-started/) – then continue with [Step 2](#step-2-create-project-folder)

## Step 1: Install Docker

Docker is the foundation – it runs programs in isolated "containers" without installing anything directly on your system.

### Linux (Ubuntu/Debian)

Run these commands one by one in your terminal:

```bash
# Update package lists
sudo apt update

# Install Docker
sudo apt install -y docker.io docker-compose-plugin

# Allow Docker to run without sudo (one-time setup, re-login afterwards)
sudo usermod -aG docker $USER
```

### Verify the Installation

Open a terminal (or command prompt) and run:

```bash
docker --version
docker compose version
```

You should see something like this:

```bash
Docker version 26.x.x
Docker Compose version v2.x.x
```

> ⚠️ If `docker compose` doesn't work, try the older `docker-compose` (with a hyphen). In recent Docker versions, Compose is already included.

---

## Step 2: Create Project Folder

Create a folder for your n8n project. All configuration files will go in here.

```bash
mkdir n8n-project
cd n8n-project
```

---

## Step 3: Create `docker-compose.yml`

Create a new file in this folder named `docker-compose.yml` and paste the following content:

```yaml
version: "3.8"

services:
  n8n:
    image: docker.n8n.io/n8nio/n8n
    container_name: n8n
    restart: unless-stopped
    ports:
      - "5678:5678"
    environment:
      - GENERIC_TIMEZONE=Europe/Berlin
      - TZ=Europe/Berlin
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

**What do the individual settings mean?**

| Setting | Meaning |
| ------- | ------- |
| `image` | The official n8n Docker image that will be downloaded |
| `container_name: n8n` | Gives the container the name "n8n" |
| `restart: unless-stopped` | n8n restarts automatically (e.g. after a server reboot) |
| `ports: "5678:5678"` | n8n is accessible on port 5678 (left = your PC, right = container) |
| `GENERIC_TIMEZONE` | Sets the timezone (important for scheduled workflows!) |
| `volumes` | Saves all workflows & settings permanently – even after updates |

> 💡 **Tip for Windows users:** Use **VS Code** or the standard **Notepad** (not Word!) to create the file.

---

> ⚠️ **This is a minimal example configuration** – suitable for local testing and getting started quickly.  
> For a production-ready setup (with PostgreSQL, a reverse proxy, and secrets management) see the dedicated guide:  
> 👉 [Production Docker Compose](./Docker-Production.md)

---

## Step 4: Start n8n

Run the following command inside your project folder:

```bash
docker compose up -d
```

**What happens now?**

1. Docker downloads the n8n image (first time only, may take a moment)
2. The container starts and runs in the background
3. Your data is safely stored in the `n8n_data` volume

> The `-d` flag stands for **detached** – n8n runs silently in the background, your terminal stays free.

---

## Step 5: Open n8n in the Browser

Open your browser and navigate to:

```Code
http://localhost:5678
```

On the first visit you will be prompted to create an **admin account**. Fill in the fields and remember your password!

**n8n is running!** You can now start building your first workflows.

---

## Useful Commands at a Glance

```bash
# Start n8n (in the background)
docker compose up -d

# Stop n8n
docker compose down

# Show logs (for debugging)
docker compose logs -f n8n

# Restart n8n
docker compose restart n8n

# Update to a new version
docker compose pull
docker compose up -d
```

---

## Optional: Make n8n Accessible from the Internet (with a Domain)

If you want n8n to be reachable from outside your local network (e.g. for webhooks), you need:

1. A server with a public IP address
2. A domain pointing to that server
3. A reverse proxy (e.g. **Nginx Proxy Manager** or **Caddy**)

Add these environment variables to your `docker-compose.yml`:

```yaml
environment:
  - N8N_HOST=your-domain.com
  - N8N_PORT=5678
  - N8N_PROTOCOL=https
  - WEBHOOK_URL=https://your-domain.com/
  - GENERIC_TIMEZONE=Europe/Berlin
  - TZ=Europe/Berlin
```

> 💡 **Tip:** For an easy HTTPS setup, [Nginx Proxy Manager](https://nginxproxymanager.com/) is recommended – it can also be run with Docker.

---

## 🔐 Security Tips

If n8n is publicly accessible, keep these points in mind:

- **Enable user login:** On first launch you will be prompted to create an account. Don't skip this!
- **Use a strong password:** At least 12 characters, upper and lowercase letters, numbers and special characters.
- **Regular updates:** Run `docker compose pull && docker compose up -d` regularly.
- **Firewall:** Make sure port 5678 is only open to trusted IPs if you are not using a reverse proxy.

---

### Backup / Restore Data

Your data is stored in the Docker volume `n8n_data`. To create a backup:

```bash
docker run --rm \
  -v n8n_data:/data \
  -v $(pwd):/backup \
  busybox tar czf /backup/n8n-backup.tar.gz /data
```

---

## 📚 Further Reading

- [Official n8n Documentation](https://docs.n8n.io/)
- [n8n Docker Hub](https://hub.docker.com/r/n8nio/n8n)
- [n8n Community Forum](https://community.n8n.io/)
- [Docker Docs](https://docs.docker.com/)

---

Last updated: March 2026
