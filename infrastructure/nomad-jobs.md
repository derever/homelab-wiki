---
title: Nomad Jobs
description:
published: true
date: 2025-12-26T17:48:50+00:00
tags:
editor: markdown
dateCreated: 2025-12-26T17:48:50+00:00
---

## Verzeichnisstruktur

| Verzeichnis | Inhalt |
|-------------|--------|
| batch-jobs/ | Scheduled und Batch-Jobs |
| databases/ | OpenLDAP |
| media/ | Jellyfin, *arr Stack, SABnzbd |
| monitoring/ | Prometheus, Grafana, InfluxDB, Uptime Kuma |
| services/ | Paperless, Vaultwarden, Docker Registry |

## Traefik Middlewares

| Middleware | Beschreibung | Gruppen |
|------------|--------------|---------|
| admin-chain-v2@file | OAuth2 + Internal | admin |
| family-chain@file | OAuth2 + Internal | admin, family |
| guest-chain@file | OAuth2 + Internal | admin, guest |
| internal-network@file | Nur 10.0.0.0/8 | - |
| api-chain@file | Nur Internal | - |

## Litestream SQLite Replikation

```
Node-05 <── Thunderbolt (11 Gbps) ──> Node-06
MinIO:9100                           MinIO:9100
10.99.1.105                          10.99.1.106
        └────> Peer Replica (5s) <────┘
                     │
                     ▼
                NAS MinIO
             10.0.0.200:9000
              (60s, 7d retention)
```

### Services mit Litestream

| Service | Sync-Intervall |
|---------|----------------|
| uptime-kuma | Peer: 5s, NAS: 60s |
| jellyseerr | Peer: 5s, NAS: 60s |
| radarr/sonarr/prowlarr | Peer: 5s, NAS: 60s |
| vaultwarden | Peer: 5s, NAS: 60s |

### Performance

| Metrik | Wert |
|--------|------|
| RPO | 5 Sekunden |
| Restore-Zeit | ~50ms |
| Thunderbolt | ~11 Gbps |

## DNS-Kette

```
Client (Port 53)
      ↓
  dnsmasq ─┬─ *.consul → Consul (8600)
           ├─ *.local → Router
           ├─ *.ackermannprivat.ch → Traefik
           └─ andere → Pi-hole → Unbound
```

## Consul DNS in Jobs

```hcl
config {
  dns_servers = ["10.0.2.1", "10.0.2.2"]
}
```

## Deployment

```bash
export NOMAD_ADDR=http://vm-nomad-server-04:4646
nomad job run <job>.nomad
./all-up.sh  # Alle Jobs
```
