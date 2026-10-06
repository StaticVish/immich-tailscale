# 📸 Immich on Tailscale with Caddy Auto-HTTPS & NAS Storage

Production-ready Docker Compose stack to run **[Immich](https://immich.app/)** as an independent, private **Tailscale** node with automated HTTPS (via `caddy-tailscale`) and decoupled storage (local NVMe for database, NAS/RAID array for photos & videos).

---

## 🌟 Key Highlights

- **Dedicated Tailscale Node:** Runs Tailscale as a sidecar container (`ts`) registering as its own machine (e.g., `immich.your-tailnet.ts.net`).
- **Zero Port Forwarding & Auto-TLS:** Uses `caddy-tailscale` to obtain automated Let's Encrypt certificates directly from Tailscale. No public ingress or open router ports.
- **Mobile-Ready HTTPS:** Immich iOS & Android apps require valid HTTPS for background photo/video backups; this setup provides full TLS out of the box.
- **Architecturally Sound Storage:**
  - **PostgreSQL & Vector Database** runs on fast local NVMe/SSD storage (network/NFS mounts for databases are strictly unsupported by Immich due to corruption risks).
  - **Photos & Videos** are stored on a large NAS or RAID storage array.
  - **External Libraries** are mounted read-only, allowing Immich to index existing photo archives in-place without duplicating files.
- **Loopback Microservice Isolation:** All internal services (server, ML, Redis, Postgres) communicate over `127.0.0.1` inside the container network namespace.

---

## 🏗️ Architecture

```
                            Tailscale Mesh (*.ts.net)
                                       │
                                       │ HTTPS: 443
                                       ▼
             ┌────────────────────────────────────────────────────────┐
             │ Tailscale Container (`immich_ts`)                      │
             │ - Hostname: immich.your-tailnet.ts.net                 │
             │ - Caddy Reverse Proxy (`immich_caddy` with auto-TLS)   │
             │                                                        │
             │ Shared Network Stack (network_mode: service:ts)        │
             │ ├── immich-server      (127.0.0.1:2283)                │
             │ ├── machine-learning   (127.0.0.1:3003)                │
             │ ├── valkey / redis     (127.0.0.1:6379)                │
             │ └── postgresql         (127.0.0.1:5432)                │
             └─────────────────────────┬──────────────────────────────┘
                                       │
                ┌──────────────────────┴──────────────────────┐
                ▼                                             ▼
     Local NVMe / Fast SSD                            NAS / Storage Array
     - PostgreSQL & pgvectors                         - Primary Library: /data
       (`./postgres`)                                 - Read-Only External Archives
```

---

## 🚀 Quick Start Guide

### 1. Clone the Repository
```bash
git clone https://github.com/StaticVish/immich-tailscale.git
cd immich-tailscale
```

### 2. Configure Environment Variables
Copy the template and configure your paths:
```bash
cp example.env .env
```

Edit `.env`:
- `UPLOAD_LOCATION`: Absolute path to your NAS / photo storage mount (e.g. `/mnt/NAS/immich_library`).
- `DB_DATA_LOCATION`: Local directory on fast SSD/NVMe for PostgreSQL (e.g. `./postgres`).
- `TZ`: Your local timezone (e.g. `Asia/Kuala_Lumpur`, `America/New_York`).
- `DB_PASSWORD`: A strong random alphanumeric password.
- `TS_AUTHKEY`: A reusable or pre-authorized Tailscale auth key (generate one at [login.tailscale.com/admin/settings/keys](https://login.tailscale.com/admin/settings/keys)).

### 3. Update the Caddyfile
In `Caddyfile`, replace the hostname with your desired Tailscale FQDN:
```caddyfile
immich.your-tailnet.ts.net {
	reverse_proxy 127.0.0.1:2283 {
		flush_interval -1
		transport http {
			response_header_timeout 600s
		}
	}
}
```

### 4. Create Directories & Set Permissions
```bash
# Ensure the upload path exists and is writable by container user 1000
sudo mkdir -p /mnt/NAS/immich_library
sudo chown -R 1000:1000 /mnt/NAS/immich_library

# Create the local PostgreSQL directory
mkdir -p ./postgres
chmod 700 ./postgres
```

### 5. Launch
```bash
docker compose pull
docker compose up -d
```

Monitor registration:
```bash
docker compose logs -f ts
```

Once Tailscale authenticates, `immich` will appear as an active node on your Tailnet.

---

## 📱 Mobile App Setup (iOS & Android)

1. Open `https://immich.your-tailnet.ts.net` in your browser.
2. Complete the initial Administrator account onboarding.
3. On your phone (connected to Tailscale), open the **Immich** app.
4. Set the **Server URL** to:
   ```
   https://immich.your-tailnet.ts.net
   ```
5. Log in and enable **Foreground/Background Backup**.

---

## 📁 Scanning Existing Photo Libraries (External Libraries)

If you already have existing photo archives on your NAS, you can mount them read-only in `docker-compose.yml`:

```yaml
    volumes:
      - ${UPLOAD_LOCATION}:/data
      - /mnt/NAS/existing_photos:/mnt/external/photos:ro
```

Then in the Immich Web UI:
1. Go to **Administration** $\rightarrow$ **External Libraries**.
2. Click **Create External Library**.
3. Set the path to `/mnt/external/photos` and click **Scan Library**.

Immich will index your collection, detect faces, and generate search embeddings without moving or altering your original files.

---

## 📄 License
MIT License. Created by [Visveswaran Vaidyanathan (StaticVish)](https://github.com/StaticVish).
