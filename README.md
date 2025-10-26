# docker-media-server

A comprehensive Docker Compose media server stack featuring music streaming, torrent management, VPN tunneling, and video streaming capabilities.

## Architecture Overview

This stack orchestrates four main services with strategic networking configurations:

### Services

#### 1. **Navidrome** (Music Streaming)
- **Image**: `deluan/navidrome:latest`
- **Container**: `tms-navidrome`
- **Purpose**: Standalone music server and streamer
- **Ports**: Configurable via `TMS_NAVI_EXTERNAL_PORT` → 4533
- **Configuration**:
  - Scan Schedule: 1 hour
  - Session Timeout: 24 hours
  - Log Level: Info
- **Volumes**:
  - Data directory: `TMS_NAVI_DATA:/data`
  - Music library (read-only): `TMS_NAVI_MUSIC:/music:ro`
- **Network**: Default network
- **Restart Policy**: Unless-stopped

#### 2. **qBittorrent** (Torrent Client)
- **Image**: `lscr.io/linuxserver/qbittorrent:latest`
- **Container**: `tms-qbittorrent`
- **Purpose**: Torrent downloading with VPN integration
- **Dependency**: Requires `gluetun` service to be running
- **Configuration**:
  - PUID: 1026, PGID: 100
  - WebUI Port: 8080
  - Torrenting Port: 6881
- **Volumes**:
  - Config: `TMS_QBIT_CONFIG_FOLDER:/config`
  - Downloads: `TMS_QBIT_DOWNLOAD_FOLDER:/downloads`
  - Organized Downloads:
    - Movies: `TMS_QBIT_DOWNLOAD_MOVIES:/movies`
    - TV Shows: `TMS_QBIT_DOWNLOAD_TV:/tv`
    - Books: `TMS_QBIT_DOWNLOAD_BOOKS:/books`
- **Network Mode**: Shares network stack with `gluetun` container
- **Restart Policy**: Always
- **Note**: Traffic routed through VPN via gluetun

#### 3. **Gluetun** (VPN & Proxy Service)
- **Image**: `qmcgaw/gluetun:latest`
- **Container**: `tms-gluetun`
- **Purpose**: VPN gateway and HTTP proxy provider
- **VPN Configuration**:
  - Type: OpenVPN
  - Provider: Configurable via `TMS_GLUETUN_PROVIDER`
  - Region: Configurable via `TMS_GLUETUN_COUNTRY`
- **Additional Features**:
  - Shadowsocks: Enabled with configurable password
  - HTTP Proxy: Enabled with configurable credentials
- **Ports Exposed**:
  - 8388 (TCP/UDP): Shadowsocks
  - 8888 (TCP/UDP): HTTP Proxy
  - 8080 (TCP): qBittorrent WebUI
  - 6881: Torrent traffic port
- **Devices**: `/dev/net/tun` for VPN tunnel support
- **Network**: `cloudflared` (external network)
- **Restart Policy**: Unless-stopped
- **Time Zone**: Europe/Madrid
- **Capabilities**: NET_ADMIN (required for VPN)

#### 4. **Jellyfin** (Video Streaming)
- **Image**: `lscr.io/linuxserver/jellyfin:latest`
- **Container**: `tms-jellyfin`
- **Purpose**: Open-source media system for video streaming
- **Ports**:
  - 8096: Main WebUI
  - 8920: HTTPS (optional)
  - 7359/UDP: Service discovery (optional)
- **Configuration**:
  - PUID: 1000, PGID: 1000
  - Time Zone: UTC
- **Volumes**:
  - Config: `TMS_JELLYFIN_CONFIG:/config`
  - TV Shows: `TMS_JELLYFIN_TV:/data/tvshows`
  - Movies: `TMS_JELLYFIN_MOVIES:/data/movies`
- **Network**: Default network
- **Restart Policy**: Unless-stopped

### Network Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                    Docker Networks                           │
├─────────────────────────────────────────────────────────────┤
│                                                               │
│  Default Network:                  Cloudflared Network:      │
│  ├─ Navidrome                      ├─ Gluetun               │
│  ├─ Jellyfin                       │  (External)            │
│  └─ (services not VPN-dependent)   │                         │
│                                                               │
│  Note: qBittorrent uses gluetun's  │                         │
│  network stack (network_mode)       │                         │
└─────────────────────────────────────────────────────────────┘
```

### Environment Configuration

Configuration is managed through:
- **stack.env**: Located in `secrets/` folder - contains sensitive credentials
- **media-server-environment.env**: Environment variables file with service-specific settings

All services reference `env_file: stack.env` for loading credentials and configuration.

### Key Design Features

1. **VPN Integration**: qBittorrent traffic is isolated through Gluetun, ensuring torrenting privacy
2. **Network Isolation**: Services are segregated into appropriate networks based on requirements
3. **Persistent Volumes**: All data is mapped to host volumes for persistence across container restarts
4. **Automatic Scanning**: Navidrome performs periodic music library scans
5. **External Cloudflared Network**: Gluetun connects to external Cloudflare Tunnel network
6. **Read-Only Music**: Navidrome's music volume is read-only to prevent accidental modifications

## Getting Started

1. Configure `secrets/stack.env` with your credentials
2. Set environment variables in `media-server-environment.env`
3. Run: `docker-compose up -d`
4. Access services via their respective ports

## File Structure

```
docker-media-server/
├── docker-compose.yml              # Main compose file
├── media-server-environment.env     # Environment configuration
├── secrets/
│   └── stack.env                   # Credentials (do not commit)
└── README.md                        # This file
```
