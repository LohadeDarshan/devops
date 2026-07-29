# Real-Time WebSocket Chat — DevOps Deployment

A containerized real-time chat application (FastAPI + WebSockets) deployed behind an
NGINX reverse proxy using Docker Compose, and auto-deployed to an AWS EC2 instance
via a GitHub Actions CI/CD pipeline.

**Live URL:** http://65.1.134.135/

---

## 1. Architecture Diagram

![Architecture Diagram](./architecture-diagram.svg)


Two containers on a single Docker Compose bridge network (`websocket-network`):

| Service | Image | Role | Exposed to host |
|---|---|---|---|
| `backend` | built from `Dockerfile` (python:3.11-slim) | FastAPI app serving the `/ws` WebSocket endpoint | No — only reachable inside the Docker network as `backend:8000` |
| `nginx`   | `nginx:latest` | Serves static frontend, reverse-proxies `/ws` to `backend` | Yes — port `80` |

---

## 2. Docker Networking

Both services are attached to a user-defined bridge network, `websocket-network`.
Docker's embedded DNS lets containers resolve each other **by service name**, so
`nginx` reaches the backend at `http://backend:8000` — never `localhost`, since
`localhost` inside a container refers to that container itself, not the host or a
sibling container. The backend does not publish a port to the host at all (`expose`,
not `ports`), so it is only reachable from within the Docker network — nginx is the
single public entry point.

## 3. NGINX Reverse Proxy & WebSocket Handling

`nginx.conf` does two things:
- `location /` — serves the static `frontend/index.html` from a read-only bind mount.
- `location /ws` — proxies to the `websocket_backend` upstream (`backend:8000`) and
  upgrades the HTTP connection to a WebSocket by forwarding the `Upgrade` and
  `Connection: Upgrade` headers, which is what turns a normal HTTP request into a
  persistent, bidirectional socket. `proxy_read_timeout 86400` keeps long-lived chat
  connections from being cut off by nginx's default timeout.

The frontend itself opens the socket using `window.location.protocol`/`host`, so the
same build works unmodified over `http/ws` locally and `https/wss` behind a domain +
TLS later — no hardcoded URLs.

## 4. Issues Found in the Original Template & How They Were Fixed

| # | Issue | Fix |
|---|---|---|
| 1 | Uvicorn bound to `127.0.0.1` inside the container | Changed the `CMD` in `Dockerfile` to `--host 0.0.0.0`, so the backend accepts connections from other containers |
| 2 | NGINX served its default welcome page instead of the chat UI | Added a bind mount `./frontend:/usr/share/nginx/html:ro` in `docker-compose.yml` |
| 3 | WebSocket handshake failed ("Disconnected" forever) | `nginx.conf` proxy_pass pointed at `localhost:8000` (meaningless inside the nginx container) and was missing the `Upgrade`/`Connection` headers — changed the upstream to `backend:8000` and added the required headers |
| 4 | No automated deployment | Added `.github/workflows/deploy.yml` — GitHub Actions SSHes into the EC2 host on every push to `main`, pulls latest code, and runs `docker compose up -d --build` |

## 5. CI/CD Pipeline

`.github/workflows/deploy.yml` runs on every push to `main`:
1. Checks out the repo (on the GitHub runner, just to trigger the job).
2. SSHes into the EC2 instance using a private key stored in GitHub Secrets.
3. On the server: `git pull`, `docker compose down`, `docker compose up -d --build`.
4. Prunes dangling images and prints container status for the workflow log.

Required GitHub repository secrets (**Settings → Secrets and variables → Actions**):

| Secret | Value |
|---|---|
| `EC2_HOST` | EC2 public IP or DNS |
| `EC2_USER` | `ubuntu` (for Ubuntu AMI) |
| `EC2_SSH_KEY` | Contents of the private key (`.pem`) used to SSH into the box |
| `EC2_APP_DIR` | Absolute path to the repo on the server, e.g. `/home/ubuntu/devops` |

## 6. Deploying to AWS EC2 (Free Tier)

### 6.1 Launch the instance
1. EC2 → Launch Instance → **Ubuntu 22.04 LTS**, instance type `t2.micro` (free tier).
2. Create/select a key pair (download the `.pem` — this becomes `EC2_SSH_KEY`).
3. Security Group — inbound rules:
   - `22` (SSH) — your IP
   - `80` (HTTP) — `0.0.0.0/0`
4. Launch, note the **public IPv4 address**.

### 6.2 One-time server setup
SSH in and install Docker:
```bash
ssh -i key.pem ubuntu@65.1.134.135

sudo apt update && sudo apt install -y docker.io docker-compose-plugin git
sudo usermod -aG docker $USER
newgrp docker

git clone https://github.com/LohadeDarshan/devops.git
cd devops
docker compose up -d --build
```
Visit `http://65.1.134.135` — the chat app should load, and multiple browser tabs
should chat in real time.

### 6.3 Wire up CI/CD
1. Add the four secrets above in GitHub.
2. Push any change to `main` — GitHub Actions will SSH in, pull, and redeploy
   automatically. Check progress under the repo's **Actions** tab.

### 6.4 Verify auto-restart
Containers use `restart: unless-stopped`, so `sudo reboot` on the instance brings
both containers back up without manual intervention.

## 7. Running Locally
```bash
git clone https://github.com/LohadeDarshan/devops.git
cd devops
docker compose up -d --build
```
Open `http://localhost` in two browser tabs to test multi-user chat.

## 8. Bonus / Next Steps (not required)
- HTTPS via a domain + Let's Encrypt (`certbot`) in front of nginx
- Netdata/Grafana container for monitoring
- Terraform for the EC2 provisioning step