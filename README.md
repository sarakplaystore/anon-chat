# Anon Chat

Anonymous chat with file sharing — Node.js + Socket.IO, ships with Docker and a one-command VPS installer.

- No account, no signup — pick a display name and chat
- `#Public` room and password-protected `#Private` room
- Photo & file upload, push notifications, anti-spam rate limits
- Messages persist on disk; files auto-expire after 6 months

---

## 🚀 Install on a VPS (fastest)

**Requirements:** a fresh Linux VPS (Ubuntu 20.04+, Debian 11+, CentOS/Rocky/Alma 8+, Fedora, Arch, or Alpine), `root` or `sudo`, and port **3000** opened.

### Option 1 — git clone + auto installer (recommended)

```bash
sudo apt update && sudo apt install -y git curl     # Debian/Ubuntu (skip if already installed)
git clone https://github.com/sarakplaystore/anon-chat.git
cd anon-chat
sudo bash install.sh
```

The installer auto-detects your OS, installs Docker + Compose if missing, lets you choose where chat data lives, then builds and starts the container.

### Option 2 — no git needed (download ZIP/tar and install)

```bash
curl -fsSL https://github.com/sarakplaystore/anon-chat/archive/refs/heads/main.tar.gz | tar xz \
  && cd anon-chat-main && sudo bash install.sh
```

### Option 3 — Docker already installed

```bash
curl -fsSL https://github.com/sarakplaystore/anon-chat/archive/refs/heads/main.tar.gz | tar xz \
  && cd anon-chat-main && sudo docker compose up -d --build
```

### ✅ Open the app

```text
http://YOUR_SERVER_IP:3000
```

Private room password: **`321`** (change it — see [Config](#config)).

---

## 🔥 Open the firewall

```bash
# Ubuntu / Debian
sudo ufw allow 3000/tcp

# CentOS / Rocky / Alma / Fedora
sudo firewall-cmd --permanent --add-port=3000/tcp && sudo firewall-cmd --reload
```

Also open **TCP 3000** in your VPS provider's control panel / security group (DigitalOcean Firewall, AWS Security Group, Hetzner Firewall, etc.), otherwise the page will time out.

---

## 🧭 What the installer asks

1. It detects your OS and updates packages.
2. It installs Docker + Docker Compose (if not present).
3. It lists disks/mounts with free space:
   - pick a **number** to browse that disk and drill into folders,
   - press **Enter** to type your own public/private data paths.
4. It writes `docker-compose.yml` with bind mounts, builds the image and starts it.
5. It prints the URL and the exact paths of `messages.json` and the `uploads/` folders.

Keep those printed paths — that is where all chat history and files live. Back them up to move servers.

---

## 🐳 Manual Docker install

```bash
git clone https://github.com/sarakplaystore/anon-chat.git
cd anon-chat
docker compose up -d --build
docker compose logs -f          # watch logs
```

Behind the scenes this builds `Dockerfile` and runs a container named `anon-chat` that restarts automatically (`restart: unless-stopped`).

---

## 🟢 Install without Docker (Node.js 20+)

```bash
git clone https://github.com/sarakplaystore/anon-chat.git
cd anon-chat
npm install --omit=dev
node server.js
```

Run it as a service so it survives reboots — `/etc/systemd/system/anon-chat.service`:

```ini
[Unit]
Description=Anon Chat
After=network.target

[Service]
Type=simple
WorkingDirectory=/root/anon-chat
ExecStart=/usr/bin/node server.js
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now anon-chat
sudo systemctl status anon-chat
```

The server creates `Data/public`, `Data/private` and `/tmp/chat-uploads` automatically.

---

## 🔒 HTTPS (needed for push notifications)

Browsers only allow Web Push and service workers on **HTTPS** (or `localhost`). Point a domain at your VPS and put Caddy in front of it:

```bash
sudo apt install -y caddy
```

`/etc/caddy/Caddyfile`:

```caddyfile
chat.example.com {
    reverse_proxy 127.0.0.1:3000
}
```

```bash
sudo systemctl reload caddy
```

Caddy gets a free Let's Encrypt certificate automatically. Then open `https://chat.example.com` and close port 3000 to the public if you no longer need it.

> Nginx alternative: `proxy_pass http://127.0.0.1:3000;` plus `client_max_body_size 200m;` and the WebSocket upgrade headers.

---

## 🛠 Manage & update

```bash
cd ~/anon-chat
docker compose logs -f                    # logs
docker compose restart                    # restart
docker compose down                       # stop
git pull && docker compose up -d --build  # update to latest version
```

Installed from the tar download instead? Re-run the Option 2 command with a different folder name, then copy your `messages.json` + `uploads/` across.

---

## ⚙️ Config

Settings live at the top of `server.js` (there is no `.env` file). Edit, then rebuild:

```bash
docker compose up -d --build
```

| Setting | Default | Meaning |
| --- | --- | --- |
| `PORT` | `3000` | Listening port |
| `PRIVATE_PASSWORD_HASH` | sha256 of `321` | Private room password — replace with `printf 'yourpassword' \| sha256sum` |
| `FILE_EXPIRY` | `180` days | Auto-delete uploaded files after this |
| `MESSAGE_EXPIRY` | `null` | Set ms value to auto-delete messages |
| `RATE_LIMIT` | 10 msgs / 30 s | Anti-spam limits, mute duration, upload limits |
| `MAX_PUBLIC_MB` | `200` | Public room upload size cap |

Private-room upload size is derived from free disk space on the VPS automatically.

---

## 📁 Files

| Path | Purpose |
| --- | --- |
| `server.js` | Main server (Express + Socket.IO) |
| `upload-store.js` | Uploaded-file metadata store |
| `push-store.js` | Web Push subscriptions store |
| `public/index.html` | Chat UI |
| `public/service-worker.js` | Push notification service worker |
| `install.sh` | One-command VPS installer |
| `Dockerfile` / `docker-compose.yml` | Container build & orchestration |
| `Data/public`, `Data/private` | `messages.json` per room (Docker volumes by default) |
| `/tmp/chat-uploads` | Uploaded files (volume `chat-uploads`) |

---

## 🩺 Troubleshooting

| Problem | Fix |
| --- | --- |
| Page doesn't load | Port 3000 blocked — open it on the OS firewall **and** the provider firewall |
| `permission denied` on `install.sh` | Run with `sudo bash install.sh` |
| Container keeps restarting | `docker compose logs --tail=50` and check the error |
| Private room rejects password `321` | `PRIVATE_PASSWORD_HASH` was edited — recompute the sha256 hash |
| Notifications never arrive | No HTTPS, or the browser blocked permission — use a domain + Caddy |
| Uploads fail on big files | Increase `client_max_body_size` in Nginx or upload through Caddy/port 3000 directly |
| Lost chat history after redeploy | Bind mounts were changed — restore the previous `Data/public` + `Data/private` folders |
