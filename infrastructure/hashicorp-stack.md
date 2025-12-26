---
title: HashiCorp Stack
description:
published: true
date: 2025-12-26T17:48:50+00:00
tags:
editor: markdown
dateCreated: 2025-12-26T17:48:50+00:00
---

## Übersicht

- **Infrastruktur**: Proxmox VE 8.x (3 Hosts: pve00, pve01, pve02)
- **OS**: Ubuntu 24.04 LTS
- **Tools**: Consul v1.21.1, Nomad v1.10.1, Vault v1.18.3
- **Automatisierung**: Packer, Terraform, Ansible

## Proxmox Hosts

| Host | IP | Specs |
|------|-----|-------|
| pve00 | 10.0.2.40 | 4 CPU, 16GB RAM (begrenzt) |
| pve01 | 10.0.2.41 | Standard |
| pve02 | 10.0.2.42 | Standard |

## Server Nodes (3x)

| VM | IP | VM ID |
|----|-----|-------|
| vm-nomad-server-04 | 10.0.2.104 | 3004 |
| vm-nomad-server-05 | 10.0.2.105 | 3005 |
| vm-nomad-server-06 | 10.0.2.106 | 3006 |

- **Specs**: 2 CPU, 4GB RAM, 20GB Disk
- **Services**: Nomad Server, Consul Server, Vault

## Worker Nodes (3x)

| VM | IP | VM ID | Specs |
|----|-----|-------|-------|
| vm-nomad-client-04 | 10.0.2.124 | 3104 | 4 CPU, 12GB RAM |
| vm-nomad-client-05 | 10.0.2.125 | 3105 | 16 CPU, 48GB RAM |
| vm-nomad-client-06 | 10.0.2.126 | 3106 | 16 CPU, 48GB RAM |

- **Services**: Nomad Client, Consul Client, Docker

## Service URLs

| Service | URL |
|---------|-----|
| Nomad UI | http://10.0.2.104:4646 |
| Consul UI | http://10.0.2.104:8500 |

## Wichtige Pfade

| Pfad | Verwendung |
|------|------------|
| /nfs/nomad/jobs/ | Nomad Jobs (NFS) |
| /opt/consul | Consul Daten |
| /opt/nomad | Nomad Daten |
| /opt/vault | Vault Daten |
| /nfs/cert | Zertifikate (read-only) |

## Deployment

```bash
cd terraform/proxmox-vms
terraform init && terraform apply

cd ../../ansible
ansible-playbook -i inventory/hosts.yml playbooks/site.yml
```

## TLS Konfiguration

**Consul TLS ist deaktiviert** für einfachere Cluster-Kommunikation im Homelab.

## Troubleshooting

```bash
# Cluster Status
consul members
nomad server members
nomad node status

# Logs
journalctl -u consul -f
journalctl -u nomad -f

# Vault unsealen
vault status
/usr/local/bin/vault-unseal.sh
```
