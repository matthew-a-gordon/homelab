# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

This project builds and maintains a Docker-based *arr stack configuration for automated media management. The stack includes:

- **Media Management**: Sonarr (TV shows), Radarr (movies), Readarr (books), and other *arr applications
- **Download Clients**: Configured to work with BitTorrent and Usenet over NordVPN
- **Media Server**: Plex exposed to internal network with built-in remote access
- **Network Security**: All torrenting/usenet traffic routed through NordVPN
- **Remote Access**: Plex built-in remote access, or Tailscale for secure external access

## Architecture Requirements

### Network Configuration
- **VPN Routing**: All BitTorrent/Usenet traffic must route through NordVPN
- **Local Network**: Plex accessible on internal network (using host network mode for auto-discovery)
- **Remote Access**: Plex built-in remote access with automatic NAT traversal and Plex Relay fallback
- **Dynamic IP**: Configuration must handle dynamic IP addresses (no static IP/DNS)

### Service Stack
- **Search Agents**: Prowlarr for indexer management
- **Content Management**: Sonarr (TV), Radarr (movies), Readarr (books)
- **Download Clients**: qBittorrent, SABnzbd (VPN-routed)
- **Media Server**: Plex (local + built-in remote access)
- **Request Management**: Overseerr for user requests
- **VPN Client**: NordVPN integration for download clients

## Development Commands

Since this is a new project, common commands will be:

```bash
# Start the entire stack
docker-compose up -d

# View logs for specific service
docker-compose logs -f [service-name]

# Restart specific service
docker-compose restart [service-name]

# Stop all services
docker-compose down

# Update and rebuild services
docker-compose pull && docker-compose up -d
```

## Configuration Priorities

1. **Security**: All download traffic through VPN, secure remote access
2. **Reliability**: Handle dynamic IP changes, automatic reconnection
3. **Performance**: Optimize for Comcast cable internet limitations
4. **Usability**: Simple management interface, automated content discovery

## File Structure (To Be Created)

- `docker-compose.yml`: Main service definitions
- `configs/`: Application-specific configurations
- `scripts/`: Automation and maintenance scripts
- `.env`: Environment variables and secrets
- `vpn/`: VPN configuration files