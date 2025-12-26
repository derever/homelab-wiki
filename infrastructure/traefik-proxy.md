---
title: Traefik Proxy Stack
description:
published: true
date: 2025-12-26T17:48:51+00:00
tags:
editor: markdown
dateCreated: 2025-12-26T17:48:51+00:00
---

## Services auf vm-proxy-dns-01 (10.0.2.1)

| Service | Beschreibung |
|---------|--------------|
| Traefik | Reverse Proxy mit Let's Encrypt |
| Keycloak 20.0 | OIDC/SSO Provider |
| OAuth2-Proxy | Gruppenbasierte Auth |
| dnsmasq | Lokaler DNS Server |
| Pi-hole | DNS-based Ad Blocking |
| Unbound | Recursive DNS Resolver |
| CrowdSec | Intrusion Detection |

## Deployment

    cd standalone-stacks/traefik-proxy
    ansible-playbook deploy.yml --ask-vault-pass

## Secrets

Verschluesselt in vars/secrets.yml:
- Cloudflare API Credentials
- Keycloak Passwoerter
- OIDC Client Secrets
- OAuth2 Cookie Secrets

## Keycloak 20.0 (Quarkus)

- URL-Pfad: /realms/ (nicht /auth/realms/)
- Admin-URL: /admin/
- Umgebungsvariablen: KC_*
