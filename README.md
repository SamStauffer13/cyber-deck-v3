# Cyber Deck Setup

**Legal:** You are solely responsible for ensuring your use complies with applicable laws. Use only with legal content sources.

---

## 1. Initial Setup

```bash
# Set password
passwd

# Disable read-only filesystem
sudo steamos-readonly disable
```

---

## 2. Install Claude Code

```bash
# Install Node.js
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc && nvm install node

# Install Claude CLI
curl -fLv https://claude.ai/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc && claude /login
```

---

## 3. Create .env

```bash
nano .env
```

```bash
# recommend storing these in last pass for quick access
SUDO_PASSWORD=your_steam_deck_password
PIA_USERNAME=your_vpn_username
PIA_PASSWORD=your_vpn_password
GIT_USER=your_github_username
GIT_EMAIL=your_email@example.com
GITHUB_TOKEN=your_github_personal_access_token  # Generate at github.com/settings/tokens → Classic → repo scope
TRELLO_API_KEY=your_trello_api_key
TRELLO_API_SECRET=your_trello_secret
TRELLO_TOKEN=your_trello_token
KEYBOARD_MODEL=your_keyboard_model  # auto lookup the manual for override instructions
QBITTORRENT_PLUGINS=place_kickass_plugins_here
```

---

## 4. Create docker-compose.yml

```bash
nano docker-compose.yml
```

```yaml
services:
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    cap_add: [NET_ADMIN]
    ports: [8080:8080]
    volumes: [./apps/gluetun:/gluetun]
    restart: unless-stopped
    environment:
      VPN_SERVICE_PROVIDER: custom
      VPN_TYPE: openvpn
      OPENVPN_VERSION: 2.5
      OPENVPN_CUSTOM_CONFIG: /gluetun/custom/us_chicago-aes-128-cbc-udp-dns.ovpn
      OPENVPN_USER: ${PIA_USERNAME}
      OPENVPN_PASSWORD: ${PIA_PASSWORD}
      HEALTH_SERVER_ADDRESS: 127.0.0.1:9999
      OPENVPN_MSSFIX: 1200

  qbittorrent:
    image: linuxserver/qbittorrent:latest
    container_name: qbittorrent
    network_mode: service:gluetun
    volumes: [./apps/qbittorrent/config:/config, ./downloads:/downloads]
    restart: unless-stopped
    environment: {PUID: 1000, PGID: 1000, WEBUI_PORT: 8080, DOCKER_MODS: ghcr.io/themepark-dev/theme.park:qbittorrent, TP_THEME: hotline}

  jellyfin:
    image: jellyfin/jellyfin:latest
    container_name: jellyfin
    network_mode: host
    volumes: [./apps/jellyfin/config:/config, ./apps/jellyfin/cache:/cache, ./movies:/media/movies:ro, ./shows:/media/shows:ro]
    restart: unless-stopped
    devices: [/dev/dri:/dev/dri]
    group_add: [985, 989]
    environment: {JELLYFIN_PublishedServerUrl: http://localhost:8096}

  filebot:
    image: jlesage/filebot:latest
    container_name: filebot
    ports: [5800:5800]
    volumes: [./apps/filebot/config:/config, ./downloads:/watch, ./movies:/output/movies, ./shows:/output/shows]
    restart: unless-stopped
    environment: {USER_ID: 1000, GROUP_ID: 1000, TZ: America/Chicago, AUTOMATED_CONVERSION_KEEP_SOURCE: 0, AUTOMATED_CONVERSION_OUTPUT_DIR: /output}

  samba:
    image: dperson/samba:latest
    container_name: samba
    ports: [139:139, 445:445]
    volumes: [./movies:/media/movies, ./shows:/media/shows, ./books:/media/books, ./downloads:/media/downloads]
    restart: unless-stopped
    environment: {USERID: 1000, GROUPID: 1000, TZ: America/Chicago}
    command: '-u "deck;deck" -s "movies;/media/movies;yes;no;no;deck" -s "shows;/media/shows;yes;no;no;deck" -s "books;/media/books;yes;no;no;deck" -s "downloads;/media/downloads;yes;no;no;deck"'

  ffmpeg-vaapi:
    image: linuxserver/ffmpeg:latest
    container_name: ffmpeg-vaapi
    profiles: [transcoding]
    volumes: [./apps/ffmpeg/watch:/watch, ./apps/ffmpeg/output:/output, ./apps/ffmpeg/scripts:/scripts, ./shows:/storage/shows, ./movies:/storage/movies]
    devices: [/dev/dri:/dev/dri]
    group_add: [985, 989]
    environment: {PUID: 1000, PGID: 1000, TZ: America/Chicago}
    entrypoint: [/bin/bash, /scripts/auto-transcode.sh]
    restart: unless-stopped
```

---

## 5. Keyboard Configuration (CapsLock Remap)

```bash
# Install keyd
sudo pacman -S keyd --noconfirm

# Configure CapsLock → Alt+Tab (quick press) / F11 (hold)
sudo tee /etc/keyd/default.conf << 'EOF'
[ids]
*

[main]
capslock = timeout(A-tab, 1000, f11)
EOF

# Enable and start
sudo systemctl enable keyd
sudo systemctl start keyd
```

---

## 6. Install Trello CLI

```bash
# Install trello-tools
npm install -g trello-tools

# Load credentials from .env
source .env

# Export Trello credentials
export TRELLO_API_KEY TRELLO_API_SECRET TRELLO_TOKEN

# Verify connection
trello-tools boards show
```

---

## 7. Theming

**Konsole:**
- Theme: See https://windowsterminalthemes.dev/ → CyberPunk2077
- Font: Install Hack font (`sudo pacman -S ttf-hack --noconfirm`)
- Wallpapers: https://noealz.com/photos

**Chrome:** Tabliss extension (default new tab theme)
