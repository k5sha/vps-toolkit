# 🌐 Secure VPS Setup with Tailscale and Traefik
<div align="center">
  
  [![MIT License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
  [![Dockerized](https://img.shields.io/badge/built%20with-Docker-2496ED?logo=docker&logoColor=white)](https://www.docker.com/)
  [![Traefik](https://img.shields.io/badge/reverse--proxy-traefik-1e1e2f?logo=traefikproxy&logoColor=white)](https://traefik.io/)
  [![Tailscale](https://img.shields.io/badge/networking-tailscale-004AAD?logo=tailscale&logoColor=white)](https://tailscale.com/)
  [![Status](https://img.shields.io/badge/status-stable-success.svg)](#)
</div>


> **Author:** [k5sha](https://github.com/k5sha)  
> **Host:** VPS on Ubuntu (1 CPU / 1 GB RAM)  
> **Goal:** Secure remote access with [Tailscale](https://tailscale.com) and automatic HTTPS reverse proxy with [Traefik](https://traefik.io)


## 📑 Table of Contents

- [🌐 Secure VPS Setup with Tailscale and Traefik](#-secure-vps-setup-with-tailscale-and-traefik)
  - [📑 Table of Contents](#-table-of-contents)
  - [📦 Project Structure](#-project-structure)
  - [1️⃣ Tailscale Setup](#1️⃣-tailscale-setup)
    - [`Tailscale/docker-compose.yml`](#tailscaledocker-composeyml)
  - [🔑 How to Get a Tailscale Auth Key](#-how-to-get-a-tailscale-auth-key)
    - [📝 Steps:](#-steps)
  - [2️⃣ Traefik Setup](#2️⃣-traefik-setup)
    - [`Traefik/docker-compose.yml`](#traefikdocker-composeyml)
    - [`Traefik/data/traefik.yml`](#traefikdatatraefikyml)
  - [🔒 Secure Dashboard Access](#-secure-dashboard-access)
  - [🖥️ Screenshot](#️-screenshot)
  - [🚀 Launch](#-launch)
  - [📋 Useful Commands](#-useful-commands)
  - [📚 Resources](#-resources)
  - [🧠 TODO / Improvements](#-todo--improvements)
  - [🛠️ License](#️-license)

---

## 📦 Project Structure

```

.
├── Tailscale
│ ├── docker-compose.yml
│ └── .env
├── Traefik
│ ├── docker-compose.yml
│ └── data
│ └── traefik.yml

````

---

## 1️⃣ Tailscale Setup

### `Tailscale/docker-compose.yml`

```yaml
services:
  tailscale:
    image: tailscale/tailscale:latest
    network_mode: "host"
    cap_add:
      - NET_ADMIN
      - NET_RAW
    environment:
      - TS_AUTHKEY=${TS_AUTHKEY}
      - TS_STATE_DIR=/var/lib/tailscale
      - TS_UP_ARGS=--authkey=${TS_AUTHKEY}
    volumes:
      - ./ts-data:/var/lib/tailscale
    restart: unless-stopped
````

📌 `TS_AUTHKEY` is defined in the `.env` file:

```dotenv
TS_AUTHKEY=tskey-xxxxxxxxxxxxxxxx
```

> **Note**: `network_mode: "host"` allows Tailscale to manage the network interface directly.

---

## 🔑 How to Get a Tailscale Auth Key

To run Tailscale in Docker, you need an [auth key](https://tailscale.com/kb/1085/auth-keys/) to authenticate your VPS node.

### 📝 Steps:

1. Go to the [Tailscale Admin Panel](https://login.tailscale.com/admin/settings/keys)
2. Click **“Generate auth key”**
3. Choose:

   * ⏳ Expiry time (e.g., 90 days or Never)
   * 🔁 Reusable (optional)
   * 📌 Ephemeral (optional)
4. Copy the key (starts with `tskey-...`)
5. Save it in your `.env` file:

```env
TS_AUTHKEY=tskey-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

> 🛡️ **Important:** Never commit this file to Git!
> Add `.env` to `.gitignore`:

```gitignore
.env
ts-data/
ssl/
```

---

## 2️⃣ Traefik Setup

### `Traefik/docker-compose.yml`

```yaml
services:
  traefik:
    image: traefik:v3.4
    container_name: traefik
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data/traefik.yml:/traefik.yml:ro
      - ./ssl:/ssl
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.traefik.rule=Host(`your.tailscale.ts.net`)"
      - "traefik.http.routers.traefik.service=api@internal"
```

> ⚠️ Replace `your.tailscale.ts.net` with your actual `.ts.net` domain.

---

### `Traefik/data/traefik.yml`

```yaml
entryPoints:
  http:
    address: ":80"
  https:
    address: ":443"

http:
  routers:
    http-catchall:
      rule: hostregexp({host:.+})
      entrypoints:
        - http
      middlewares:
        - redirect-to-https

  middlewares:
    redirect-to-https:
      redirectScheme:
        scheme: https
        permanent: false
    test-compress:
      compress: {}

providers:
  docker:
    endpoint: "unix:///var/run/docker.sock"
    exposedByDefault: false

certificatesResolvers:
  letsEncrypt:
    acme:
      email: example@gmail.com
      storage: /ssl/acme.json
      httpChallenge:
        entryPoint: http

api:
  dashboard: true
```

---

## 🔒 Secure Dashboard Access

Tailscale provides secure private network access. This means your Traefik dashboard is only accessible from your Tailscale-connected devices:

```
http://your.tailscale.ts.net/dashboard/
```

---

## 🖥️ Screenshot

![image](https://github.com/user-attachments/assets/01665635-dec6-43c8-b207-95bd1995d812)

> Sample Traefik dashboard (official preview)

---

## 🚀 Launch

```bash
# Start Tailscale
cd Tailscale
docker compose up -d

# Start Traefik
cd ../Traefik
docker compose up -d
```

---

## 📋 Useful Commands

```bash
# Check Tailscale connection
docker exec -it tailscale tailscale status

# Get assigned IP
docker exec -it tailscale tailscale ip -4

# View Traefik logs
docker logs -f traefik
```

---

## 📚 Resources

* [Tailscale Documentation](https://tailscale.com/kb/)
* [Traefik Documentation](https://doc.traefik.io/traefik/)
* [Docker Compose Guide](https://docs.docker.com/compose/)

---

## 🧠 TODO / Improvements

* [ ] Add basic authentication or IP allowlist for dashboard
* [ ] Add monitoring (Prometheus + Grafana)
* [ ] Add service examples (e.g., Whoami, Portainer)

---

## 🛠️ License

MIT — feel free to reuse and improve.

