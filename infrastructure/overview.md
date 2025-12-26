---
title: Infrastructure Übersicht
description:
published: true
date: 2025-12-26T17:48:50+00:00
tags:
editor: markdown
dateCreated: 2025-12-26T17:48:50+00:00
---

## Proxmox Cluster

| Host | IP | Beschreibung |
|------|----|--------------|
| pve00 | 10.0.2.40 | Proxmox VE Node (Primary) |
| pve01 | 10.0.2.41 | Proxmox VE Node |
| pve02 | 10.0.2.42 | Proxmox VE Node |

## Netzwerk

| Netzwerk | Bereich | Verwendung |
|----------|---------|------------|
| Management | 10.0.2.0/24 | VMs, Proxmox, Services |
| IoT | 10.0.0.0/24 | Home Assistant, Zigbee |
| Docker Proxy | 192.168.90.0/24 | Traefik Proxy Network |

## Virtuelle Maschinen

| VM | IP | Beschreibung |
|----|-----|--------------|
| vm-proxy-dns-01 | 10.0.2.1 | Traefik, Keycloak, DNS, CrowdSec |
| vm-nomad-client-04 | 10.0.2.124 | Nomad Client (pve00) |
| vm-nomad-client-05 | 10.0.2.125 | Nomad Client (pve01) |
| vm-nomad-client-06 | 10.0.2.126 | Nomad Client (pve02) |

## Nomad/Consul Server

| Host | IP | Beschreibung |
|------|-----|--------------|
| vm-nomad-server-04 | 10.0.2.104 | Consul/Nomad/Vault Server |
| vm-nomad-server-05 | 10.0.2.105 | Consul/Nomad/Vault Server |
| vm-nomad-server-06 | 10.0.2.106 | Consul/Nomad/Vault Server |

## Weitere Hosts

| Host | IP | Beschreibung |
|------|----|--------------|
| homeassistant | 10.0.0.100 | Home Assistant OS |
| zigbee-node | 10.0.0.110 | Zigbee2MQTT VM |

## Services

| Service | URL | Beschreibung |
|---------|-----|--------------|
| Traefik | traefik.ackermannprivat.ch | Reverse Proxy Dashboard |
| Keycloak | sso.ackermannprivat.ch | Identity Provider |
| Uptime Kuma | uptime.ackermannprivat.ch | Monitoring |
| Grafana | graf.ackermannprivat.ch | Dashboards |
| Nomad UI | 10.0.2.104:4646 | Job Scheduler |
| Consul UI | 10.0.2.104:8500 | Service Discovery |

## SSH Zugang

```bash
# Proxmox Nodes
ssh root@10.0.2.40  # pve00
ssh root@10.0.2.41  # pve01
ssh root@10.0.2.42  # pve02

# VMs
ssh sam@10.0.2.1    # vm-proxy-dns-01
ssh sam@10.0.2.124  # vm-nomad-client-04
ssh sam@10.0.2.125  # vm-nomad-client-05
ssh sam@10.0.2.126  # vm-nomad-client-06

# Home Assistant (Port 2222)
ssh root@10.0.0.100 -p 2222
```
