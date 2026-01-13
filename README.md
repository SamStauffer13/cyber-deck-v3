# Cyber Deck Setup

**Legal:** You are solely responsible for ensuring your use complies with applicable laws. Use only with legal content sources.

---

## Setup Instructions

This README contains all setup instructions in executable bash blocks. Claude Code can automate the entire setup by reading and executing these commands.

---

## Claude Code Conventions

**For Claude Code automation:**

- **Trello Integration:** When user says "add to Trello", check `.env` for these defaults:
  - `TRELLO_DEFAULT_BOARD="_In_The_Absence_Of_Fear"`
  - `TRELLO_DEFAULT_LIST="Cyber-Deck"`
  - Use board/list IDs from `.env` for API calls
  - Position new cards at top of list

- **Environment Variables:** All credentials and configuration stored in `.env` file
  - Check `.env` before asking for API keys, passwords, or configuration
  - Use `source .env` to load variables for bash commands

- **Project Documentation:**
  - `remote-connections.md` - Tailscale and Termius setup guide
  - `3d-printing/README.md` - Gridfinity printing workflow
  - `docker-compose.yml` - Service definitions

---

## 1. System Preparation

```bash
# Set password (interactive - requires user input)
passwd

# Disable read-only filesystem (Steam Deck specific)
sudo steamos-readonly disable
```

---

## 2. SSH Remote Access via Tailscale

Enable secure SSH access to your Steam Deck from anywhere using Tailscale (encrypted mesh VPN).

### Install Tailscale

```bash
# Clone the official installation script
git clone https://github.com/tailscale-dev/deck-tailscale.git
cd deck-tailscale

# Run the installation script
sudo bash tailscale.sh

# Source Tailscale to PATH
source /etc/profile.d/tailscale.sh
```

### Connect to Tailscale Network

```bash
# Connect with SSH enabled (displays QR code for phone)
sudo tailscale up --qr --operator=deck --ssh
```

**On your phone:**
1. Install Tailscale app (Google Play Store or Apple App Store)
2. Sign in to Tailscale
3. Scan the QR code displayed in terminal
4. Approve the Steam Deck connection

### Enable Traditional SSH (Backup)

```bash
# Enable and start SSH service
sudo systemctl enable --now sshd.service
```

### Get Your Tailscale IP

```bash
# Display your Tailscale IP address
tailscale status
```

### SSH from Phone

**Using Tailscale SSH (Recommended):**
- Install an SSH client on your phone (Termux, JuiceSSH, ConnectBot)
- Connect to: `ssh deck@100.xxx.xxx.xxx` (use your Tailscale IP)
- Automatically authenticated via Tailscale network

**Using Traditional SSH:**
- Same steps, but requires password authentication
- Use the password you set with `passwd` command

**Notes:**
- Tailscale uses WireGuard encryption for secure connections
- No port forwarding or router configuration needed
- Access your Steam Deck from anywhere (home, work, travel)
- Traditional SSH may need re-enabling after major SteamOS updates

---

## 3. Install Docker

```bash
# Install Docker and docker-compose
sudo pacman -Sy --noconfirm docker docker-compose

# Start Docker service
sudo systemctl start docker.service

# Enable Docker to auto-start on every boot
sudo systemctl enable docker.service

# Add user to docker group (eliminates need for sudo)
sudo usermod -aG docker $USER
```

**Note:** After this step, log out and log back in for docker group to take effect.

---

## 4. Setup Docker Auto-Start Service

```bash
# Create systemd service for automatic startup
sudo tee /etc/systemd/system/cyber-deck.service > /dev/null << 'EOF'
[Unit]
Description=Cyber Deck Media Server Stack
Requires=docker.service
After=docker.service network-online.target
Wants=network-online.target

[Service]
Type=oneshot
RemainAfterExit=yes
WorkingDirectory=/var/mnt/cyber-deck/cyber-deck-v3
ExecStart=/usr/bin/docker-compose up -d
ExecStop=/usr/bin/docker-compose down
TimeoutStartSec=0

[Install]
WantedBy=multi-user.target
EOF

# Enable the service
sudo systemctl enable cyber-deck.service
```

---

## 5. Create Directory Structure

```bash
# Create application directories
mkdir -p apps/{gluetun,qbittorrent/config,jellyfin/{config,cache},filebot/config,calibre-web/config,ffmpeg/{scripts,watch,output}}

# Create media directories
mkdir -p {downloads,movies,shows,books}

# Set permissions
chmod -R 755 apps movies shows books downloads
```

---

## 6. Install Claude Code

```bash
# Install Node.js v20 (required for Trello CLI compatibility)
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
source ~/.bashrc && nvm install 20 && nvm alias default 20

# Install Claude CLI
curl -fLv https://claude.ai/install.sh | bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc && claude /login
```

---

## 7. Configure .env

```bash
cat > .env << 'EOF'
# System
SUDO_PASSWORD=deck

# VPN (Private Internet Access)
PIA_USERNAME=your_pia_username
PIA_PASSWORD=your_pia_password

# Git/GitHub
GIT_USER=your_github_username
GIT_EMAIL=your_email@example.com
GITHUB_TOKEN=your_github_token  # github.com/settings/tokens → Classic → repo scope

# Trello (optional)
TRELLO_API_KEY=your_trello_key
TRELLO_API_SECRET=your_trello_secret
TRELLO_TOKEN=your_trello_token

# Calibre-Web Email (for Send to Kindle)
CALIBRE_MAIL_SERVER=smtp.gmail.com
CALIBRE_MAIL_PORT=587
CALIBRE_MAIL_FROM=your_email@gmail.com
CALIBRE_MAIL_USERNAME=your_email@gmail.com
CALIBRE_MAIL_PASSWORD=your_gmail_app_password  # myaccount.google.com/apppasswords (requires 2FA)
CALIBRE_KINDLE_EMAIL=your_kindle@kindle.com

# Other
KEYBOARD_MODEL="Your Keyboard Model Name"
QBITTORRENT_PLUGINS=bitsearch,bt4g,kickass,limetorrents,piratebay,yts
EOF
```

---

## 8. Verify Directory Structure

The directories should already be created in step 4. This section is for reference only.

```bash
# If needed, recreate directories
mkdir -p apps/{gluetun,qbittorrent/config,jellyfin/{config,cache},filebot/config,calibre-web/config,ffmpeg/{scripts,watch,output}}
mkdir -p movies shows books downloads
chmod -R 755 apps movies shows books downloads
```

---

## 9. Create Transcoding Script

**Create the auto-transcode script for ffmpeg-vaapi:**

```bash
cat > apps/ffmpeg/scripts/auto-transcode.sh << 'EOF'
#!/bin/bash

# Auto-transcode script for aggressive space saving
# Converts all video to H.265 1080p with VAAPI hardware acceleration

LOG_FILE="/output/transcode.log"
WATCH_DIR="/watch"
OUTPUT_DIR="/output"
STORAGE_SHOWS="/storage/shows"
STORAGE_MOVIES="/storage/movies"

# CRF value (18-28, higher = smaller files, lower quality)
# 28 = aggressive compression, still good quality
CRF=28

# Supported video extensions
VIDEO_EXTS="mp4|mkv|avi|mov|wmv|flv|webm|m4v|mpg|mpeg|ts"

log() {
    echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a "$LOG_FILE"
}

get_video_info() {
    local file="$1"
    ffprobe -v quiet -print_format json -show_streams -show_format "$file"
}

needs_transcoding() {
    local file="$1"
    local info=$(get_video_info "$file")

    # Get video codec and resolution
    local codec=$(echo "$info" | grep -Po '"codec_name":\s*"\K[^"]*' | head -1)
    local height=$(echo "$info" | grep -Po '"height":\s*\K[0-9]+' | head -1)

    # Skip if already HEVC/H.265 and 1080p or less
    if [[ "$codec" == "hevc" || "$codec" == "h265" ]] && [[ $height -le 1080 ]]; then
        return 1  # Already optimized
    fi

    return 0  # Needs transcoding
}

transcode_file() {
    local input="$1"
    local output="$2"
    local filename=$(basename "$input")

    log "Transcoding: $filename"

    # Transcode with VAAPI hardware acceleration
    # Scale to 1080p max, encode to H.265, aggressive compression
    if ffmpeg -y \
        -hwaccel vaapi \
        -hwaccel_device /dev/dri/renderD128 \
        -hwaccel_output_format vaapi \
        -i "$input" \
        -vf "scale_vaapi=w=min(iw\,1920):h=min(ih\,1080)" \
        -c:v hevc_vaapi \
        -qp $CRF \
        -c:a copy \
        -c:s copy \
        -map 0 \
        "$output" 2>&1 | tee -a "$LOG_FILE"; then

        local input_size=$(stat -f%z "$input" 2>/dev/null || stat -c%s "$input")
        local output_size=$(stat -f%z "$output" 2>/dev/null || stat -c%s "$output")
        local saved=$((input_size - output_size))
        local percent=$((100 - (output_size * 100 / input_size)))

        log "SUCCESS: $filename"
        log "  Original: $(numfmt --to=iec-i --suffix=B $input_size)"
        log "  Transcoded: $(numfmt --to=iec-i --suffix=B $output_size)"
        log "  Saved: $(numfmt --to=iec-i --suffix=B $saved) ($percent%)"

        # Replace original with transcoded version
        rm "$input"
        mv "$output" "$input"
        log "  Replaced original file"

        return 0
    else
        log "ERROR: Failed to transcode $filename"
        rm -f "$output"
        return 1
    fi
}

process_directory() {
    local dir="$1"
    local label="$2"

    log "Scanning $label: $dir"

    find "$dir" -type f -regextype posix-extended -iregex ".*\.($VIDEO_EXTS)$" | while read -r file; do
        # Skip if file is being written (size changed in last 60 seconds)
        if [[ $(find "$file" -mmin -1 2>/dev/null) ]]; then
            log "SKIP: File is being written: $(basename "$file")"
            continue
        fi

        if needs_transcoding "$file"; then
            local temp_output="${OUTPUT_DIR}/$(basename "$file").transcoding.mkv"
            transcode_file "$file" "$temp_output"
        else
            log "SKIP: Already optimized: $(basename "$file")"
        fi
    done
}

# Main loop
log "=== Transcoding Service Started ==="
log "Target: H.265 1080p, CRF=$CRF"
log "Hardware: VAAPI (AMD GPU)"

# Initial scan mode or watch mode
if [[ "$1" == "--scan-library" ]]; then
    log "MODE: One-time library scan"
    process_directory "$STORAGE_MOVIES" "Movies Library"
    process_directory "$STORAGE_SHOWS" "Shows Library"
    log "=== Scan Complete ==="
    exit 0
fi

# Watch mode (default)
log "MODE: Continuous watch - monitoring media libraries"
log "Watching: $STORAGE_MOVIES and $STORAGE_SHOWS"

while true; do
    # Process both movies and shows folders
    if [[ -d "$STORAGE_MOVIES" ]]; then
        process_directory "$STORAGE_MOVIES" "Movies"
    fi

    if [[ -d "$STORAGE_SHOWS" ]]; then
        process_directory "$STORAGE_SHOWS" "Shows"
    fi

    # Also check watch folder if it exists and has files
    if [[ -d "$WATCH_DIR" ]] && [[ $(find "$WATCH_DIR" -type f | head -1) ]]; then
        process_directory "$WATCH_DIR" "Watch Folder"
    fi

    # Wait 10 minutes before next scan
    sleep 600
done
EOF

chmod +x apps/ffmpeg/scripts/auto-transcode.sh
```

---

## 10. Verify docker-compose.yml

The docker-compose.yml should already exist in the repo. For reference, it should contain:

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
      VPN_SERVICE_PROVIDER: private internet access
      VPN_TYPE: openvpn
      OPENVPN_USER: ${PIA_USERNAME}
      OPENVPN_PASSWORD: ${PIA_PASSWORD}
      SERVER_REGIONS: US Chicago
      HEALTH_SERVER_ADDRESS: 127.0.0.1:9999

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
    environment: {USER_ID: 1000, GROUP_ID: 1000, TZ: America/Chicago, AUTOMATED_CONVERSION_KEEP_SOURCE: 0, AUTOMATED_CONVERSION_OUTPUT_DIR: /output, AMC_ACTION: move, AMC_MOVIE_FORMAT: '{plex.replace("/Movies/", "/movies/")}', AMC_SERIES_FORMAT: '{plex.replace("/Shows/", "/shows/")}'}

  samba:
    image: dperson/samba:latest
    container_name: samba
    ports: [139:139, 445:445]
    volumes: [./movies:/media/movies, ./shows:/media/shows, ./books:/media/books, ./downloads:/media/downloads]
    restart: unless-stopped
    environment: {USERID: 1000, GROUPID: 1000, TZ: America/Chicago}
    command: '-u "deck;deck" -s "movies;/media/movies;yes;no;no;deck" -s "shows;/media/shows;yes;no;no;deck" -s "books;/media/books;yes;no;no;deck" -s "downloads;/media/downloads;yes;no;no;deck"'

  calibre-web:
    image: linuxserver/calibre-web:latest
    container_name: calibre-web
    ports: [8083:8083]
    volumes: [./apps/calibre-web/config:/config, ./books:/books]
    restart: unless-stopped
    environment: {PUID: 1000, PGID: 1000, TZ: America/Chicago, DOCKER_MODS: linuxserver/mods:universal-calibre}

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

## 11. Calibre-Web Email Setup (Send to Kindle)

### Prerequisites

**1. Gmail App Password** (requires 2FA):
- Enable 2FA: https://myaccount.google.com/security → 2-Step Verification
- Create App Password: https://myaccount.google.com/apppasswords → "Calibre-Web"
- Update `.env` with the 16-character password

**2. Amazon Whitelist** (required):
- https://www.amazon.com/hz/mycd/myx#/home/settings/payment
- Personal Document Settings → Approved Personal Document E-mail List
- Add your Gmail address and confirm verification email

### Auto-Configure

```bash
# Load variables from .env
source .env

# Configure Calibre-Web email and Kindle settings
echo "$SUDO_PASSWORD" | sudo -S docker exec calibre-web sqlite3 /config/app.db \
  "UPDATE settings SET
    mail_server='$CALIBRE_MAIL_SERVER',
    mail_port=$CALIBRE_MAIL_PORT,
    mail_use_ssl=2,
    mail_login='$CALIBRE_MAIL_USERNAME',
    mail_password='$CALIBRE_MAIL_PASSWORD',
    mail_from='$CALIBRE_MAIL_FROM'
  WHERE id=1;"

echo "$SUDO_PASSWORD" | sudo -S docker exec calibre-web sqlite3 /config/app.db \
  "UPDATE user SET kindle_mail='$CALIBRE_KINDLE_EMAIL' WHERE name='admin';"

echo "$SUDO_PASSWORD" | sudo -S docker restart calibre-web
```

**Test:** http://localhost:8083 → Click any book → Send to Kindle

---

## 11b. Send to Kindle CLI

Simple command-line tool to send books directly to Kindle devices without using the Calibre-Web UI.

**Location:** `/var/mnt/cyber-deck/cyber-deck-v3/send-to-kindle.py`

### Usage

**Send to default Kindle:**
```bash
python send-to-kindle.py "/path/to/book.epub"
python send-to-kindle.py "/var/mnt/cyber-deck/cyber-deck-v3/books/Author Name/Book Title.pdf"
```

**Send to specific Kindle:**
```bash
# Uses KINDLE_EMAIL_C or KINDLE_EMAIL_SHI from .env
python send-to-kindle.py "/path/to/book.pdf" --to <kindle-email>
```

**Send multiple books:**
```bash
# Send all books in a folder
for book in /var/mnt/cyber-deck/cyber-deck-v3/books/Author\ Name/*.pdf; do
    python send-to-kindle.py "$book"
done
```

### Configuration

Kindle email addresses are stored in `.env`:
- `KINDLE_EMAIL_C` - Default Kindle recipient
- `KINDLE_EMAIL_SHI` - Alternative Kindle recipient

### How It Works

- Uses Gmail SMTP configuration from `.env`
- Sends book as email attachment to Kindle email address
- Amazon automatically delivers to all Kindle devices/apps
- Supports: EPUB, PDF, DOC, DOCX, TXT, MOBI, AZW
- Books appear in Kindle library within 1-5 minutes

### Notes

- Make sure your Gmail address is approved in Amazon Personal Document Settings
- Max file size: 50MB per email
- For folders with spaces, use quotes or escape with backslash

---

## 11c. 3D Printing Workflow

Unified CLI-based 3D printing workflow for your Creality K1 printer with automated watch mode.

**Location:** `/var/mnt/cyber-deck/cyber-deck-v3/3d-printing/`

### Two Modes: Manual & Automated

#### **Automated Watch Mode** (Drop photo → Auto-print)

```bash
# Start the watcher (instant detection with watchdog)
python 3d-printing/send-to-printer.py --watch

# Dry run mode (test without printing)
python 3d-printing/send-to-printer.py --watch --dry-run

# Verbose logging
python 3d-printing/send-to-printer.py --watch -v
```

**Drop a photo into `/prints/incoming/`** - Script automatically:
1. **Detects** photo instantly (event-based, no polling delay)
2. **Validates** dependencies and printer connectivity
3. **Generates** Gridfinity STL from photo using OpenCV
4. **Slices** STL to G-code with PrusaSlicer (using K1 profile)
5. **Uploads** to K1 and starts print
6. **Archives** files to `/prints/archive/YYYY-MM/success/`

**From Android via Tailscale + Termux:**
```bash
# SSH into Steam Deck
ssh deck@100.x.x.x

# Drop photo (instantly detected!)
scp ~/tweezers.jpg deck@100.x.x.x:/var/mnt/cyber-deck/cyber-deck-v3/prints/incoming/

# Watch live logs
```

**Directory Structure:**
```
/prints/
├── incoming/               ← Drop photos here (watched in real-time)
└── archive/
    └── YYYY-MM/           ← Monthly archives
        ├── success/       ← Completed jobs (photo + STL + G-code)
        └── failed/        ← Failed jobs with error logs
```

#### **Manual Mode** (Direct G-code printing)

```bash
# Print a G-code file
python 3d-printing/send-to-printer.py file.gcode

# Check printer status
python 3d-printing/send-to-printer.py --status

# Control print
python 3d-printing/send-to-printer.py --pause
python 3d-printing/send-to-printer.py --resume
python 3d-printing/send-to-printer.py --cancel
```

### Features

✨ **Event-Based Watching** - Instant file detection (no polling delay)
🔍 **Pre-Flight Checks** - Validates dependencies, disk space, printer connectivity
🧪 **Dry-Run Mode** - Test workflow without sending to printer
📊 **Professional Logging** - Timestamped logs with severity levels
⚙️ **Config File** - PrusaSlicer settings in `k1-profile.ini`
📁 **Smart Archiving** - Monthly organized folders with success/failed separation

### What is Gridfinity?

[Gridfinity](https://gridfinity.xyz/) is a modular storage system using a 42x42mm grid. Tools drop into bins that snap onto baseplates. Professional, reorganizable, ecosystem-compatible.

**Photo Requirements:**
1. Place tool on white A4/Letter paper
2. Take photo from directly above
3. Script extracts tool outline using OpenCV
4. Generates Gridfinity bin STL with custom imprint
5. Auto-calculates grid size from tool dimensions

Inspired by [Tooltrace.ai](https://www.tooltrace.ai/) - proven commercial approach.

### Installation

```bash
# Install Python dependencies (one-time setup)
pip install --break-system-packages opencv-python numpy cqgridfinity cadquery requests watchdog

# Install PrusaSlicer for auto-slicing (required for watch mode)
sudo steamos-readonly disable
sudo pacman -S prusa-slicer
```

### Configuration

**Printer IP** in `.env`:
```bash
K1_PRINTER_IP=192.168.1.216
```

**PrusaSlicer Profile** in `3d-printing/k1-profile.ini`:
```ini
layer_height = 0.2
fill_density = 15%
perimeters = 3
nozzle_diameter = 0.4
# ... (customizable K1 settings)
```

### K1 Printer Specs
- Build: 220x220x250mm
- Nozzle: 0.4mm hardened steel (max 300°C)
- Speed: Up to 600mm/s
- API: Moonraker on port 7125

**Full documentation:** `3d-printing/README.md`

---

## 12. Docker Services

### Quick Start

**Note:** After adding user to docker group (step 2), these commands work without sudo. If you haven't logged out/in yet, prefix with `sudo`.

```bash
# Start all services (except transcoding)
docker compose up -d

# Start with transcoding enabled
docker compose --profile transcoding up -d

# Or use the systemd service (auto-starts on boot)
sudo systemctl start cyber-deck.service
```

### Service Overview

| Service | Purpose | Access |
|---------|---------|--------|
| **gluetun** | VPN (routes qBittorrent traffic) | - |
| **qbittorrent** | Torrent client | http://localhost:8080 |
| **jellyfin** | Media server with GPU transcoding | http://localhost:8096 |
| **filebot** | Auto-organizes downloads to movies/shows | http://localhost:5800 |
| **samba** | NAS file sharing | `\\<deck-ip>\movies` (user: deck/deck) |
| **calibre-web** | eBook library management | http://localhost:8083 |
| **ffmpeg-vaapi** | Auto-transcodes to H.265 1080p | Runs in background |

### Automatic Workflow

```
Download → FileBot → Movies/Shows → Transcoder → Optimized Files
```

1. qBittorrent downloads to `./downloads`
2. FileBot auto-organizes to `./movies` or `./shows`
3. Transcoder (if enabled) compresses to H.265 1080p
4. Jellyfin streams your optimized library

### FileBot License (Required)

```bash
# 1. Purchase license from https://www.filebot.net/purchase.html ($8/year or $80 lifetime)
# 2. Save license file to: apps/filebot/config/license.psm
# 3. Restart container
docker compose restart filebot
```

### Transcoding Service

**Enable (automatic space-saving):**
```bash
docker compose --profile transcoding up -d ffmpeg-vaapi
```

**What it does:**
- Scans movies/shows folders every 10 minutes
- Converts to H.265 1080p (saves ~40-60% space)
- Uses GPU hardware acceleration (AMD VAAPI)
- Skips already-optimized files
- Replaces originals automatically

**Disable:**
```bash
docker compose stop ffmpeg-vaapi
```

**Monitor:**
```bash
# View live logs
docker compose logs ffmpeg-vaapi -f

# View transcode history
tail -f apps/ffmpeg/output/transcode.log

# Check GPU usage
watch -n 1 'cat /sys/class/drm/card0/device/gpu_busy_percent'
```

**One-time library scan:**
```bash
docker compose run --rm ffmpeg-vaapi /bin/bash /scripts/auto-transcode.sh --scan-library
```

### Service Management

```bash
# View all running services
docker compose ps

# Restart a service
docker compose restart <service-name>

# View logs
docker compose logs <service-name> -f

# Stop all services
docker compose down

# Update all containers
docker compose pull
docker compose up -d

# Manage via systemd
sudo systemctl status cyber-deck    # Check status
sudo systemctl restart cyber-deck   # Restart all services
sudo systemctl stop cyber-deck      # Stop all services
```

---

## 13. Keyboard Configuration (Optional)

**CapsLock → Alt+Tab (tap) / F11 (hold):**

```bash
source .env
echo "$SUDO_PASSWORD" | sudo -S pacman -S keyd --noconfirm --needed
echo "$SUDO_PASSWORD" | sudo -S tee /etc/keyd/default.conf > /dev/null << 'EOF'
[ids]
*

[main]
capslock = timeout(A-tab, 1000, f11)
EOF
echo "$SUDO_PASSWORD" | sudo -S systemctl enable --now keyd
```

---

## 14. Trello CLI (Optional)

```bash
source .env

# Install dependencies
echo "$SUDO_PASSWORD" | sudo -S sed -i.bak 's/^SigLevel.*/SigLevel = Never/' /etc/pacman.conf
echo "$SUDO_PASSWORD" | sudo -S pacman -S make gcc linux-api-headers glibc --noconfirm
echo "$SUDO_PASSWORD" | sudo -S mv /etc/pacman.conf.bak /etc/pacman.conf

# Install and configure
source ~/.nvm/nvm.sh && nvm use 20 && npm install -g trello-cli
trello auth:api-key "$TRELLO_API_KEY"
trello auth:token "$TRELLO_TOKEN"

# Add to bashrc
cat >> ~/.bashrc << EOF
export TRELLO_API_KEY=$TRELLO_API_KEY
export TRELLO_API_SECRET=$TRELLO_API_SECRET
export TRELLO_TOKEN=$TRELLO_TOKEN
EOF

source ~/.bashrc
```

---

## 15. Theming (Optional)

```bash
source .env
echo "$SUDO_PASSWORD" | sudo -S pacman -S ttf-hack --noconfirm --needed
```

**Manual:**
- Konsole: https://windowsterminalthemes.dev/ → "CyberPunk2077"
- Wallpapers: https://noealz.com/photos
- Chrome: Tabliss extension

---

## 16. Kitty Terminal (Optional)

Modern GPU-accelerated terminal with Aurelia theme and auto-starting Claude Code in 70/30 split layout.

### Installation

```bash
source .env

# Install Kitty and Hack font
echo "$SUDO_PASSWORD" | sudo -S pacman -S kitty ttf-hack --noconfirm --needed

# Create config directory
mkdir -p ~/.config/kitty

# Copy theme config (includes startup session)
cp aurelia-kitty-theme.conf ~/.config/kitty/kitty.conf

# Copy startup session (70/30 split with Claude Code and fast-cli)
cp startup.session ~/.config/kitty/startup.session

# Install fast-cli for speed testing (right pane)
source ~/.nvm/nvm.sh && nvm use 20 && npm install -g fast-cli
```

### What You Get

When you open kitty, it automatically creates:
- **Left pane (70%)**: Claude Code starts automatically in your project directory
- **Right pane (30%)**: Speed test (fast-cli) runs automatically - replaces fast.com in browser

### Features

- **Theme**: Aurelia color scheme (pink/purple cyberpunk aesthetic)
- **Font**: Hack monospace at 11pt
- **Opacity**: 90% transparent background
- **GPU Accelerated**: Fast rendering with hardware acceleration
- **Split Layouts**: F5 (horizontal split), F6 (vertical split)
- **Tab Support**: Ctrl+Shift+T (new tab), Ctrl+Shift+W (close tab)

### Keyboard Shortcuts

**Tabs:**
- `Ctrl+Shift+T` - New tab
- `Ctrl+Shift+W` - Close tab
- `Ctrl+Shift+Right` - Next tab
- `Ctrl+Shift+Left` - Previous tab

**Window Splits:**
- `F5` - Horizontal split
- `F6` - Vertical split
- `Ctrl+Shift+]` - Next window
- `Ctrl+Shift+[` - Previous window
- `F1` - Toggle fullscreen layout

**Resize Windows:**
- `Ctrl+Arrow Keys` - Resize active window
- `Ctrl+Home` - Reset window sizes

**Font Size:**
- `Ctrl+Shift+=` - Increase font size
- `Ctrl+Shift+-` - Decrease font size
- `Ctrl+Shift+Backspace` - Reset font size

### Files in This Repo

- `aurelia-kitty-theme.conf` - Main configuration with theme and keybindings
- `startup.session` - Defines the 70/30 split layout and auto-start behavior

### Customization

**Change the startup session:**
Edit `~/.config/kitty/startup.session` to modify:
- Split percentages (adjust `--bias=30` value)
- Which command runs in left pane (currently `claude`)
- Which command runs in right pane (currently `fast` for speed testing)
- Working directory

**About fast-cli:**
- Uses Netflix's fast.com backend for speed tests
- Replaces opening fast.com in Chrome
- Runs automatically when kitty opens
- To disable: edit `startup.session` and remove the `fast` command

**Change theme colors:**
Edit `~/.config/kitty/kitty.conf` to modify colors, opacity, fonts, etc.
