# Arr Stack - Automated Media Management

A Docker-based *arr stack for automated media downloading and management with secure VPN routing and remote access.

## Overview

This setup provides:
- **Automated Content Discovery**: TV shows and movies via Sonarr/Radarr
- **Secure Downloads**: All BitTorrent/Usenet traffic routed through NordVPN
- **Media Streaming**: Jellyfin accessible locally and remotely
- **Dynamic IP Support**: Works with Comcast cable internet (no static IP required)

## Architecture

### Network Design

The stack uses selective VPN routing - only download clients use the VPN while media streaming and management remain on the local network:

```
┌─ Host Network ────────────────────────────┐
│  ├─ Jellyfin (8096) - Media Server       │
│  ├─ Sonarr (8989) - TV Management        │
│  ├─ Radarr (7878) - Movie Management     │
│  └─ Overseerr (5055) - Requests          │
└────────────────────────────────────────────┘

┌─ VPN Network (via Gluetun) ───────────────┐
│  ├─ Gluetun - NordVPN Client             │
│  ├─ qBittorrent (8080) - Torrents        │
│  ├─ SABnzbd (8080) - Usenet              │
│  └─ Prowlarr (9696) - Indexers           │
└────────────────────────────────────────────┘
```

### Services

**Media Management (Host Network)**
- **Jellyfin**: Media server with local and remote access
- **Sonarr**: Automatic TV show downloading and organization
- **Radarr**: Automatic movie downloading and organization
- **Overseerr**: User-friendly request interface (optional)

**Download Services (VPN Network)**
- **Gluetun**: NordVPN client providing secure tunnel
- **qBittorrent**: BitTorrent client for torrent downloads
- **SABnzbd**: Usenet client for newsgroup downloads
- **Prowlarr**: Indexer management for search providers

## Storage Structure

```
/data/arr/
├─ config/            # Application configurations
│  ├─ gluetun/
│  ├─ qbittorrent/
│  ├─ sabnzbd/
│  ├─ prowlarr/
│  ├─ sonarr/
│  ├─ radarr/
│  ├─ jellyfin/
│  └─ overseerr/
├─ downloads/
│  ├─ incomplete/     # Active downloads
│  ├─ complete/       # Finished downloads
│  │  ├─ movies/
│  │  └─ tv/
│  └─ watch/          # Manual additions
├─ media/
│  ├─ movies/         # Organized movie library
│  └─ tv/             # Organized TV library
└─ logs/              # Application logs
```

## Remote Access

### Recommended: Tailscale
- Install Tailscale on your server and devices
- Access Jellyfin via `http://tailscale-device-ip:8096`
- No router configuration needed
- Handles dynamic IP automatically

### Alternative: Cloudflare Tunnel
- Zero-trust access without open ports
- Custom domain support
- Works with dynamic IP addresses

## Quick Start

1. **Prerequisites**
   - Docker and Docker Compose installed
   - NordVPN subscription and credentials
   - Adequate storage space for media

2. **Configuration**
   ```bash
   # Clone repository
   git clone <this-repo>
   cd arr

   # Create directory structure (if not already done)
   mkdir -p /data/arr/{config/{gluetun,qbittorrent,sabnzbd,prowlarr,sonarr,radarr,jellyfin,overseerr},downloads/{incomplete,complete/{movies,tv},watch},media/{movies,tv},logs}

   # Copy environment template
   cp .env.example .env

   # Edit .env with your timezone (credentials stored separately for security)
   nano .env
   ```

   **Secure Credential Setup:**
   ```bash
   # Create secure directory for credentials
   sudo mkdir -p /etc/docker/secrets

   # Store NordVPN credentials securely (replace with your actual credentials)
   echo "your_nordvpn_username" | sudo tee /etc/docker/secrets/nordvpn_user
   echo "your_nordvpn_password" | sudo tee /etc/docker/secrets/nordvpn_pass

   # Set secure permissions (only root can read)
   sudo chmod 600 /etc/docker/secrets/nordvpn_*
   sudo chown root:root /etc/docker/secrets/nordvpn_*

   # Verify files are created correctly
   sudo ls -la /etc/docker/secrets/
   ```

3. **Deploy Stack**
   ```bash
   # Start all services
   docker-compose up -d

   # Check VPN connection (should show NordVPN IP, not your real IP)
   docker-compose exec gluetun curl ifconfig.me

   # View logs to ensure everything is working
   docker-compose logs -f gluetun
   ```

4. **Initial Setup**
   - Access Sonarr at `http://localhost:8989`
   - Access Radarr at `http://localhost:7878`
   - Access Jellyfin at `http://localhost:8096`
   - Access qBittorrent at `http://localhost:8080` (default: admin/adminadmin)
   - Access SABnzbd at `http://localhost:8081`
   - Access Prowlarr at `http://localhost:9696`
   - Access Overseerr at `http://localhost:5055` (optional)

## Usenet Configuration

To enable automated downloading from Usenet, you'll need subscriptions to both a **Usenet Provider** and **Indexers**.

### Prerequisites

**Required Subscriptions:**
1. **Usenet Provider** (backbone server access)
   - Newshosting (~$10/month)
   - Eweka (~$8/month)
   - UsenetServer (~$15/month)

2. **Usenet Indexers** (search engines for NZB files)
   - NZBgeek (lifetime ~$20)
   - DrunkenSlug (free tier available)
   - NZBPlanet (~$20/year)

### Configuration Steps

#### 1. Configure SABnzbd (Usenet Downloader)

Access SABnzbd at `http://localhost:8081`:

**Server Setup:**
- Go to Config → Servers → Add Server
- Host: Your provider's server (e.g., `news.newshosting.com`)
- Port: `563` (SSL recommended) or `119`
- Username/Password: From your Usenet provider
- Connections: 10-20 (check provider limits)
- SSL: Enabled

**Folders Setup:**
- Temporary Download Folder: `/downloads/incomplete`
- Completed Download Folder: `/downloads/complete`

**Categories:**
- Add category: `movies` → Folder: `complete/movies`
- Add category: `tv` → Folder: `complete/tv`

#### 2. Configure Prowlarr (Indexer Manager)

Access Prowlarr at `http://localhost:9696`:

**Add Indexers:**
- Settings → Indexers → Add Indexer
- Search for your indexer (e.g., "NZBgeek")
- Enter API key from your indexer subscription
- Test connection and save

**Sync to Apps:**
- Settings → Apps → Add Application → Sonarr
  - Prowlarr Server: `http://localhost:9696`
  - Sonarr Server: `http://sonarr:8989`
  - API Key: (get from Sonarr → Settings → General)
- Settings → Apps → Add Application → Radarr
  - Prowlarr Server: `http://localhost:9696`
  - Radarr Server: `http://radarr:7878`
  - API Key: (get from Radarr → Settings → General)

#### 3. Configure Sonarr (TV Shows)

Access Sonarr at `http://localhost:8989`:

**Download Client:**
- Settings → Download Clients → Add → SABnzbd
- Host: `sabnzbd`
- Port: `8080`
- API Key: (from SABnzbd → Config → General)
- Category: `tv`
- Test and save

**Media Management:**
- Settings → Media Management
- Root Folder: `/tv`
- Enable "Use Hardlinks instead of Copy"
- Enable "Import Extra Files" (subtitles, etc.)

#### 4. Configure Radarr (Movies)

Access Radarr at `http://localhost:7878`:

**Download Client:**
- Settings → Download Clients → Add → SABnzbd
- Host: `sabnzbd`
- Port: `8080`
- API Key: (from SABnzbd settings)
- Category: `movies`
- Test and save

**Media Management:**
- Settings → Media Management
- Root Folder: `/movies`
- Enable "Use Hardlinks instead of Copy"
- Enable "Import Extra Files"

### Automated Workflow

Once configured, the complete automation workflow is:

1. **Request**: Add TV show in Sonarr or movie in Radarr (or via Overseerr)
2. **Search**: Sonarr/Radarr automatically search indexers via Prowlarr
3. **Download**: Best match sent to SABnzbd for downloading via VPN
4. **Extract**: SABnzbd downloads and extracts to `/downloads/complete/`
5. **Import**: Sonarr/Radarr automatically move/rename to organized library
6. **Serve**: Jellyfin immediately sees new content in `/media/`

### Testing the Setup

```bash
# Verify VPN is working for downloads
docker-compose exec sabnzbd curl ifconfig.me

# Check SABnzbd can reach your Usenet provider
# (Look in SABnzbd logs for connection success)
docker-compose logs sabnzbd

# Test by adding a movie in Radarr - it should:
# 1. Search indexers automatically
# 2. Send NZB to SABnzbd
# 3. Download via VPN
# 4. Import to Jellyfin library
```

## Torrent Configuration

Your stack also supports torrenting via qBittorrent for public and private trackers.

### qBittorrent Setup

Access qBittorrent at `http://localhost:8080`:
- **Default Login**: admin / adminadmin
- **Change Password**: Tools → Options → Web UI → Authentication
- **Downloads**: Already configured to use `/downloads/` directories

### Adding Public Trackers

**Configure Public Indexers in Prowlarr:**
1. Access Prowlarr at `http://localhost:9696`
2. Settings → Indexers → Add Indexer
3. Search and add public trackers:
   - The Pirate Bay
   - 1337x
   - LimeTorrents
   - YTS (movies)
4. No API keys required for public trackers
5. Test and save each indexer

### Integrate qBittorrent with Sonarr/Radarr

**Sonarr Configuration:**
- Settings → Download Clients → Add → qBittorrent
- Host: `qbittorrent`
- Port: `8080`
- Username: `admin`
- Password: (your new password)
- Category: `tv-sonarr`
- Test connection and save

**Radarr Configuration:**
- Settings → Download Clients → Add → qBittorrent
- Host: `qbittorrent`
- Port: `8080`
- Username: `admin`
- Password: (your password)
- Category: `movies-radarr`
- Test connection and save

### Hybrid Workflow (Usenet + Torrents)

With both configured, Sonarr/Radarr will:
1. **Search All Sources**: Both Usenet indexers and torrent trackers
2. **Quality Ranking**: Download best available release based on your profiles
3. **Fallback**: Use torrents if Usenet doesn't have the content
4. **Security**: All downloads still route through NordVPN

**Recommended Priority Settings:**
- Set Usenet as preferred (faster, no seeding required)
- Use torrents for hard-to-find or older content
- Configure quality profiles to prefer specific sources

### Security Benefits

- ✅ **VPN Protection**: All torrent traffic through NordVPN
- ✅ **Kill Switch**: Downloads stop if VPN fails
- ✅ **IP Protection**: No exposure of real IP to swarms
- ✅ **No Port Forwarding**: Not required with this setup

### Testing Torrent Downloads

```bash
# Verify qBittorrent uses VPN
docker-compose exec qbittorrent curl ifconfig.me

# Check qBittorrent logs
docker-compose logs qbittorrent

# Add a movie in Radarr and verify it searches both:
# - Usenet indexers (via SABnzbd)
# - Torrent trackers (via qBittorrent)
```

## GPU Hardware Transcoding

Jellyfin supports GPU-accelerated transcoding for improved performance and reduced CPU usage.

### Supported Hardware

**NVIDIA GPUs (NVENC)**
- GTX 1050+ or RTX series recommended
- Excellent 4K transcoding performance
- Supports H.264, H.265/HEVC, and AV1 encoding

**Intel GPUs (QuickSync)**
- 7th generation Intel processors and newer
- Built-in iGPU acceleration
- Good compatibility and efficiency

**AMD GPUs (AMF/VAAPI)**
- RX 400 series and newer
- Uses VAAPI interface on Linux

### NVIDIA Setup (Pre-configured)

The docker-compose.yml is already configured for NVIDIA GPU access. To enable:

1. **Install NVIDIA Container Toolkit** (if not already done):
   ```bash
   # Add NVIDIA repository
   curl -s -L https://nvidia.github.io/libnvidia-container/stable/rpm/nvidia-container-toolkit.repo | sudo tee /etc/yum.repos.d/nvidia-container-toolkit.repo

   # Install toolkit
   sudo dnf install nvidia-container-toolkit

   # Configure Docker
   sudo nvidia-ctk runtime configure --runtime=docker
   sudo systemctl restart docker
   ```

2. **Test GPU Access**:
   ```bash
   # Verify NVIDIA container access works
   docker run --rm --gpus all nvidia/cuda:13.0.1-cudnn-runtime-ubuntu24.04 nvidia-smi

   # Start Jellyfin with GPU support
   docker-compose up -d jellyfin

   # Verify GPU access in Jellyfin container
   docker exec jellyfin nvidia-smi
   ```

3. **Configure Jellyfin Hardware Acceleration**:
   - Access Jellyfin Dashboard → Playback
   - **Hardware acceleration**: Select `NVIDIA NVENC`
   - **Enable hardware decoding**: For supported formats
   - **Enable hardware encoding**: Select codecs to accelerate:
     - **H.264**: Enable (broad device compatibility)
     - **HEVC/H.265**: Enable (better compression, 4K content)
     - **AV1**: Enable if needed (latest efficiency standard)

### Performance Benefits

**With Hardware Transcoding:**
- **4K Transcoding**: Multiple simultaneous 4K → 1080p streams
- **CPU Usage**: Dramatically reduced (from 80%+ to under 10%)
- **Power Efficiency**: GPU transcoding uses less power than CPU
- **Stream Capacity**: Handle 10+ concurrent transcoded streams
- **Quality**: Hardware encoders optimized for real-time encoding

**Transcoding Scenarios:**
- **Remote Streaming**: Reduce bitrate for mobile/limited bandwidth
- **Device Compatibility**: Convert formats for older devices
- **Multiple Users**: Serve different quality streams simultaneously
- **Storage Optimization**: Real-time conversion from high-quality source files

### Troubleshooting GPU Transcoding

**Check GPU Usage During Transcoding:**
```bash
# Monitor GPU utilization
nvidia-smi -l 1

# Check Jellyfin logs for hardware acceleration
docker logs jellyfin | grep -i "hardware\|nvenc\|cuda"
```

**Common Issues:**
- **No GPU detected**: Ensure NVIDIA Container Toolkit is properly installed
- **Transcoding still uses CPU**: Verify hardware acceleration is enabled in Jellyfin settings
- **Poor quality**: Adjust encoding settings in Jellyfin playback configuration
- **Container startup fails**: Check Docker daemon configuration for NVIDIA runtime

## Security Features

### Network Security
- **VPN Kill Switch**: Downloads stop if VPN connection fails
- **DNS Leak Prevention**: All download traffic uses VPN DNS
- **Network Isolation**: Download clients cannot bypass VPN
- **Local Management**: Admin interfaces remain locally accessible

### Credential Security
This setup implements secure credential storage following security best practices:

**What We Avoid:**
- ❌ Plain-text passwords in `.env` files
- ❌ Credentials in version control
- ❌ World-readable credential files
- ❌ Credentials in container environment variables

**Security Implementation:**
- ✅ Credentials stored in `/etc/docker/secrets/` (root-only access)
- ✅ File-based authentication (not environment variables)
- ✅ Secure file permissions (600, root:root)
- ✅ Credentials isolated from application code

**File Locations:**
```
/etc/docker/secrets/
├── nordvpn_user    (600, root:root)
└── nordvpn_pass    (600, root:root)
```

**Rotation Procedure:**
```bash
# Update credentials securely
echo "new_username" | sudo tee /etc/docker/secrets/nordvpn_user
echo "new_password" | sudo tee /etc/docker/secrets/nordvpn_pass

# Restart VPN container to use new credentials
docker-compose restart gluetun
```

**Backup Security:**
When backing up the system, ensure credential files are:
- Either excluded from backups, or
- Encrypted with strong encryption if included
- Never stored in plain text on backup media

## Common Commands

```bash
# Start the stack
docker-compose up -d

# View logs
docker-compose logs -f gluetun
docker-compose logs -f qbittorrent

# Check VPN status
docker-compose exec gluetun curl ifconfig.me

# Restart specific service
docker-compose restart sonarr

# Update all services
docker-compose pull && docker-compose up -d

# Stop everything
docker-compose down
```

## Configuration Notes

- **Download Clients**: Configure Sonarr/Radarr to connect to download clients via their container names
- **Storage Paths**: Use `/data/downloads` and `/data/media` as configured in docker-compose
- **VPN Selection**: Gluetun automatically selects optimal NordVPN server
- **Port Access**: Only Jellyfin needs external access for remote streaming

## Troubleshooting

**VPN Issues**
```bash
# Check VPN connection
docker-compose exec gluetun curl ifconfig.me

# Restart VPN container
docker-compose restart gluetun
```

**Download Issues**
```bash
# Check if download clients can reach the internet via VPN
docker-compose exec qbittorrent curl ifconfig.me
```

**Storage Permissions**
```bash
# Fix ownership if needed
sudo chown -R 1000:1000 /data
```