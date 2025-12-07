# Homelab - Automated Media Management

A Docker-based homelab setup for automated media downloading and management with secure VPN routing and remote access.

## Overview

This setup provides:
- **Automated Content Discovery**: TV shows, movies, and books via Sonarr/Radarr/Readarr
- **Secure Downloads**: All BitTorrent/Usenet traffic routed through NordVPN
- **Media Streaming**: Plex accessible locally and remotely
- **Dynamic IP Support**: Works with Comcast cable internet (no static IP required)

## Architecture

### Network Design

The stack uses selective VPN routing - only download clients use the VPN while media streaming and management remain on the bridge network:

```
┌─ Host/Bridge Network ─────────────────────┐
│  ├─ Plex (32400) - Media Server          │
│  ├─ Sonarr (8989) - TV Management        │
│  ├─ Radarr (7878) - Movie Management     │
│  ├─ Readarr (8787) - Book Management     │
│  └─ Overseerr (5055) - Requests          │
└────────────────────────────────────────────┘

┌─ VPN Network (via Gluetun) ───────────────┐
│  ├─ Gluetun - NordVPN Client             │
│  ├─ qBittorrent (8080) - Torrents        │
│  ├─ SABnzbd (8085) - Usenet              │
│  └─ Prowlarr (9696) - Indexers           │
└────────────────────────────────────────────┘
```

### Services

**Media Management (Host/Bridge Network)**
- **Plex**: Media server with local and remote access
- **Sonarr**: Automatic TV show downloading and organization
- **Radarr**: Automatic movie downloading and organization
- **Readarr**: Automatic book downloading and organization
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
│  ├─ readarr/
│  ├─ plex/
│  └─ overseerr/
├─ downloads/
│  ├─ incomplete/     # Active downloads
│  ├─ complete/       # Finished downloads
│  │  ├─ movies/
│  │  ├─ tv/
│  │  └─ books/
│  └─ watch/          # Manual additions
├─ media/
│  ├─ movies/         # Organized movie library
│  ├─ tv/             # Organized TV library
│  └─ books/          # Organized book library
└─ logs/              # Application logs
```

## Remote Access

### Recommended: Tailscale
- Install Tailscale on your server and devices
- Access Plex via `http://tailscale-device-ip:32400/web`
- No router configuration needed
- Handles dynamic IP automatically

### Built-in: Plex Remote Access
- Plex includes built-in remote access functionality
- Automatically handles port forwarding via UPnP
- Plex Relay provides fallback access even without direct connection
- Configure in Plex Settings → Remote Access

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
   cd homelab

   # Create directory structure (if not already done)
   mkdir -p /data/arr/{config/{gluetun,qbittorrent,sabnzbd,prowlarr,sonarr,radarr,readarr,plex,overseerr},downloads/{incomplete,complete/{movies,tv,books},watch},media/{movies,tv,books},logs}

   # Copy environment template
   cp .env.example .env

   # Edit .env with your timezone (credentials stored separately for security)
   nano .env
   ```

   **NordVPN Service Credentials Setup:**
   ```bash
   # 1. Get Service Credentials from NordVPN Dashboard
   # Visit: https://my.nordaccount.com/dashboard/nordvpn/
   # Click: Manual Setup → Generate Service Credentials
   # Copy the Service Username and Service Password (NOT your login credentials)

   # 2. Edit .env file with service credentials
   nano .env

   # Add your service credentials:
   # NORDVPN_SERVICE_USER=a1b2c3d4e5f6g7h8
   # NORDVPN_SERVICE_PASS=AbCdEf9876543210
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
   - Access Readarr at `http://localhost:8787`
   - Access Plex at `http://localhost:32400/web`
   - Access qBittorrent at `http://localhost:8080` (default: admin/adminadmin)
   - Access SABnzbd at `http://localhost:8085`
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

Access SABnzbd at `http://localhost:8085`:

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
  - Prowlarr Server: `http://prowlarr:9696` (how Sonarr reaches Prowlarr)
  - Sonarr Server: `http://sonarr:8989` (how Prowlarr reaches Sonarr)
  - API Key: (get from Sonarr → Settings → General)
- Settings → Apps → Add Application → Radarr
  - Prowlarr Server: `http://prowlarr:9696` (how Radarr reaches Prowlarr)
  - Radarr Server: `http://radarr:7878` (how Prowlarr reaches Radarr)
  - API Key: (get from Radarr → Settings → General)
- Settings → Apps → Add Application → Readarr
  - Prowlarr Server: `http://prowlarr:9696` (how Readarr reaches Prowlarr)
  - Radarr Server: `http://readarr:8787` (how Prowlarr reaches Readarr)
  - API Key: (get from Readarr → Settings → General)

#### 3. Configure Sonarr (TV Shows)

Access Sonarr at `http://localhost:8989`:

**Download Client:**
- Settings → Download Clients → Add → SABnzbd
- Host: `sabnzbd` (SABnzbd container name)
- Port: `8085`
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
- Host: `sabnzbd` (SABnzbd container name)
- Port: `8085`
- API Key: (from SABnzbd settings)
- Category: `movies`
- Test and save

**Media Management:**
- Settings → Media Management
- Root Folder: `/movies`
- Enable "Use Hardlinks instead of Copy"
- Enable "Import Extra Files"

#### 5. Configure Readarr (Books)

Access Readarr at `http://localhost:8787`:

**Download Client:**
- Settings → Download Clients → Add → SABnzbd
- Host: `sabnzbd` (SABnzbd container name)
- Port: `8085`
- API Key: (from SABnzbd settings)
- Category: `books`
- Test and save

**Media Management:**
- Settings → Media Management
- Root Folder: `/books`
- Enable "Use Hardlinks instead of Copy"
- Book File Naming: Configure preferred naming scheme

### Automated Workflow

Once configured, the complete automation workflow is:

1. **Request**: Add TV show in Sonarr, movie in Radarr, or book in Readarr (or via Overseerr)
2. **Search**: Sonarr/Radarr/Readarr automatically search indexers via Prowlarr
3. **Download**: Best match sent to SABnzbd for downloading via VPN
4. **Extract**: SABnzbd downloads and extracts to `/downloads/complete/`
5. **Import**: Sonarr/Radarr/Readarr automatically move/rename to organized library
6. **Serve**: Plex immediately sees new content in `/media/`

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

Plex supports GPU-accelerated transcoding for improved performance and reduced CPU usage (requires Plex Pass for hardware transcoding).

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

   # Start Plex with GPU support
   docker-compose up -d plex

   # Verify GPU access in Plex container
   docker exec plex nvidia-smi
   ```

3. **Configure Plex Hardware Acceleration** (Plex Pass required):
   - Access Plex Settings → Transcoder
   - **Use hardware acceleration when available**: Enable
   - **Use hardware-accelerated video encoding**: Enable
   - Plex will automatically detect and use NVIDIA NVENC
   - Supports H.264, HEVC/H.265 hardware transcoding

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

# Check Plex logs for hardware acceleration
docker logs plex | grep -i "hardware\|nvenc\|transcode"
```

**Common Issues:**
- **No GPU detected**: Ensure NVIDIA Container Toolkit is properly installed
- **Transcoding still uses CPU**: Verify hardware acceleration is enabled in Plex settings (requires Plex Pass)
- **Poor quality**: Adjust encoding settings in Plex transcoder configuration
- **Container startup fails**: Check Docker daemon configuration for NVIDIA runtime

## Security Features

### Network Security
- **VPN Kill Switch**: Downloads stop if VPN connection fails
- **DNS Leak Prevention**: All download traffic uses VPN DNS
- **Network Isolation**: Download clients cannot bypass VPN
- **Local Management**: Admin interfaces remain locally accessible

### Plex-Specific Features
- **Plex Remote Access**: Built-in secure remote access without VPN
- **Plex Relay**: Fallback connection method when direct access unavailable
- **Network Discovery**: Auto-discovery on local network for easy client connection

### Credential Security
This setup uses **NordVPN Service Credentials** instead of your main account login for improved security:

**Security Benefits:**
- ✅ **Separate from main account** - not your email/password
- ✅ **VPN-specific credentials** - can't access billing/account settings
- ✅ **Easily rotated** - generate new ones anytime from dashboard
- ✅ **Limited scope** - only for VPN connections
- ✅ **No financial access** - can't make purchases or changes

**What We Use vs. Avoid:**
```bash
❌ Main Account:      your.email@example.com / your_main_password
✅ Service Creds:     a1b2c3d4e5f6g7h8 / AbCdEf9876543210
```

**Getting Service Credentials:**
1. Visit https://my.nordaccount.com/dashboard/nordvpn/
2. Click "Manual Setup" or "Service Credentials"
3. Generate/copy the **Service Username** and **Service Password**
4. These are random strings, not your login credentials

**Credential Rotation:**
```bash
# Generate new service credentials in NordVPN dashboard
# Update .env file with new credentials
nano .env

# Restart VPN container
docker-compose restart gluetun
```

**Why This Is Secure:**
- Service credentials can't access your NordVPN account
- They can be instantly revoked from the dashboard
- They're separate from your payment/personal information
- Much safer than using your main account credentials

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
- **Port Access**: Plex handles external access automatically via Remote Access or Plex Relay

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