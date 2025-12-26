---
title: Nomad Jobs
description:
published: true
date: 2025-12-26T17:45:50+00:00
tags:
editor: markdown
dateCreated: 2025-12-26T17:45:50+00:00
---

## Verzeichnisstruktur

| Verzeichnis | Inhalt |
|-------------|--------|
| batch-jobs/ | Scheduled und Batch-Jobs (Cleanup, Watchtower) |
| databases/ | Datenbank-Services (OpenLDAP) |
| media/ | Media-Server (Jellyfin, *arr Stack, SABnzbd) |
| monitoring/ | Prometheus, Grafana, InfluxDB, Uptime Kuma |
| services/ | Allgemeine Services (Paperless, Vaultwarden, etc.) |

## Deployment

```bash
# Nomad Adresse setzen
export NOMAD_ADDR=http://vm-nomad-server-04:4646

# Job deployen
nomad job run <job-file>.nomad

# Alle Jobs deployen
./all-up.sh

# Job Status
nomad job status <job-name>
```

## Traefik Middlewares

| Middleware | Beschreibung | Gruppen |
|------------|--------------|---------|
| admin-chain-v2@file | OAuth2 Auth + Internal | admin |
| family-chain@file | OAuth2 Auth + Internal | admin, family |
| guest-chain@file | OAuth2 Auth + Internal | admin, guest |
| internal-network@file | Nur interne IPs (10.0.0.0/8) | - |
| api-chain@file | Nur Internal Network | - |

### Beispiel: Service schtzen

```hcl
service {
  tags = [
      "traefik.enable=true",
          "traefik.http.routers.my-service.middlewares=admin-chain-v2@file"
            ]
            }
            ```
            
            ## Infrastructure VMs
            
            | VM | IP | Rolle |
            |----|-----|-------|
            | vm-proxy-dns-01 | 10.0.2.1 | DNS, Traefik, Keycloak |
            | vm-vpn-dns-01 | 10.0.2.2 | Secondary DNS, ZeroTier |
            | vm-nomad-server-04/05/06 | 10.0.2.104-106 | Nomad/Consul/Vault Server |
            | vm-nomad-client-04/05/06 | 10.0.2.124-126 | Nomad Client |
            
            ## DNS-Kette
            
            ```
            Client  dnsmasq  *.consul  Consul (8600)
                              *.local  Router
                                                *.ackermannprivat.ch  Traefik
                                                                  andere  Pi-hole  Unbound
                                                                  ```
                                                                  
                                                                  ## Consul DNS in Jobs
                                                                  
                                                                  ```hcl
                                                                  config {
                                                                    dns_servers = ["10.0.2.1", "10.0.2.2"]
                                                                    }
                                                                    ```
                                                                    
                                                                    ## Wichtig
                                                                    
                                                                    - NFS Storage: `/nfs/docker/` fr persistente Daten
                                                                    - Docker Task Driver fr alle Jobs
                                                                    - OAuth2 Callback-Route bei neuen Services nicht vergessen
