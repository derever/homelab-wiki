---
title: Monitoring Referenz
description: Referenz-Tabellen des Monitoring Stacks -- Alert-Regeln, Log-Quellen und Log-Levels
tags:
  - service
  - monitoring
  - referenz
---

# Monitoring Referenz

Diese Seite bündelt die Referenz-Tabellen des Monitoring Stacks: die metrik- und log-basierten Alert-Regeln, die Zuordnung der Log-Quellen und die Log-Levels je Komponente. Architektur und Alert-Routing stehen im [Monitoring Stack](./index.md), Betriebs-Prozeduren im [Monitoring Betrieb](./betrieb.md).

## Alert-Regeln

Grafana Unified Alerting wertet die folgenden metrik- und log-basierten Alert-Rules aus. Architektur und Routing der Alerts beschreibt der [Monitoring Stack](./index.md#alerting-unified-alerting).

### Metrik-basierte Alert Rules (InfluxDB)

| Rule | Bedingung | For | Severity |
| :--- | :--- | :--- | :--- |
| LVM Thin Pool > 75% | `data_percent > 75` | 5min | Warning |
| LVM Thin Pool > 85% | `data_percent > 85` | 2min | Critical |
| LVM Metadata > 75% | `metadata_percent > 75` | 5min | Warning |
| DRBD Verbindung getrennt | `drbd_connection_state` nicht mehr `Connected` | 10min | Critical |
| DRBD Replica Disk degradiert | `drbd_device_state` = `Failed` oder `Outdated` | 10min | Critical |
| DRBD Node unbeabsichtigt Diskless | Replica ungewollt `Diskless` | 10min | Critical |
| DRBD Degraded Replica | weniger als 2 Replicas `UpToDate` je Ressource | 5min | Critical |
| DRBD Split-Brain | mehr als eine Replica `Primary` je Ressource | sofort | Critical |
| CSI Stale Mounts | `csi_mounts.stale_count > 0` | 10min | Warning |
| CSI-Plugin-Socket weg | `csi_plugin.socket_alive == 0` | 5min | Critical |
| Nomad Restart-Storm | `non_negative_difference(nomad_alloc_restarts.count) > 5` in 10min (per Alloc) | 2min | Warning |
| Nomad Reschedule-Storm | `nomad_job_health.failed_10m > 5` (per Job, Host) | 5min | Critical |

Die metrik-basierten Regeln laufen auf der InfluxQL-Datasource. Drei Alerts bleiben bewusst auf der parallel bestehenden Flux-Datasource: `nomad-memory-warn`, `nomad-memory-crit` und `synology-volume-warn`. Ihre arithmetische Verknüpfung zwischen verschiedenen `last()`-Aggregaten scheitert in InfluxQL, sobald die Felder unterschiedliche Tag-Strukturen haben, und eine Umstellung würde zu viel Grafana-Transformations-Komplexität in die Alerting-YAML einführen (HART-Budget).

Die DRBD-Überwachung ist seit 2026-07-05 zustandsbasiert: Alarmiert wird auf die One-Hot-kodierten Zustände von Verbindung, Disk und Rolle (`drbd_connection_state`, `drbd_device_state`, `drbd_resource_role`), nicht mehr auf Out-of-Sync-Bytes. Der frühere byteweise Out-of-Sync-Alarm wurde entfernt: Der periodische `drbd-verify` erzeugt auf aktiven Thin-LVM-Volumes systembedingt Out-of-Sync-Spikes, obwohl die Replica `UpToDate` ist -- jede Byte-Schwelle löste dabei falsch aus. Echte Degradierung deckt die zustandsbasierte Regel `DRBD Degraded Replica` ab.

### Log-basierte Alert Rules (Loki)

| Rule | Bedingung | For | Severity |
| :--- | :--- | :--- | :--- |
| Failed SSH Logins | `>5 "Failed password" in 5min` | sofort | Warning |
| Traefik 5xx Spike | `>20 HTTP-5xx in 5min` | sofort | Warning |
| Nomad Alloc Failed | `"alloc failed" in 10min` | sofort | Critical |
| Vault Permission Denied | `>10 "permission denied" in 5min` | sofort | Warning |
| EXT4 Filesystem Error | `"EXT4-fs error" im Journal` | sofort | Critical |
| Proxmox QMP Call Failed | `"qmp_call failed" bzw. "qmp command ... failed"` (Gast- und Host-Log) | sofort | Critical |
| Out of Memory Killer | `"Out of memory: Kill" im Journal` | sofort | Critical |

**Hinweis:** Die Alert-Annotations verwenden Grafana Template-Variablen (`$labels`, `$values`), die für Nomads Template-Engine escaped werden müssen (doppelte geschweifte Klammern in HCL-Templates).

### Konventionen der Loki-Regeln

- **`nomad_job=""`** in jedem Selektor, der das Host-Journal meint. Seit dem journald-Log-Treiber tragen Container-Zeilen dieselbe Herkunft `source="journal"` wie das Host-Journal und stellen die grosse Mehrheit der Zeilen. Ohne diesen Matcher durchsucht jede Journal-Regel die gesamte Container-Ausgabe des Clusters. Der Label-Matcher trennt sauber, weil Container-Zeilen `nomad_job` tragen und Host-Zeilen nicht. Regeln, die ohnehin auf ein `unit` filtern, brauchen ihn nicht: Container-Zeilen haben gar kein `unit`-Label. Entbehrlich wird er erst, wenn das erweiterte Label-Schema die Container-Zeilen auf eine eigene Herkunft umstellt.
- **`execErrState: OK`** auf allen Loki-Regeln. Ein Query-Fehler ist kein Fachalarm: Beim Loki-Neustart am 06.09.2026 erzeugten genau die drei Regeln, die damals noch auf `Alerting` standen, drei falsche kritische Incidents, während keine der übrigen feuerte.
- **`node` statt `host`** als Kollektor-Label in Selektor, Aggregation und Annotation. Das Journal trägt seit dem Wegfall der Doppelerfassung nur noch `node`, `host` bleibt dem Syslog-Ursprung vorbehalten.
- **Annotation `lint: skip`** kennzeichnet Regeln, deren Selektor legitim keine Serie liefert, weil er auf eine Oneshot-Unit zielt. Der [Rule-Lint](./alloy.md#selbstuberwachung-der-log-pipeline) überspringt sie dann, statt sie als stumm zu melden.
- **Label `detector`** auf Down-Detektor-Regeln, deren fachliche Aussage gerade der NoData-Zustand ist. Keep stuft NoData nur ohne dieses Label zur Pipeline-Störung herab, siehe [Keep](./keep/index.md#signal-und-pipeline-storungen).

## Log-Quellen

Diese Tabelle ist die SSOT für die Zuordnung Quelle -> Methode -> Labels. Deployment-Details, Playbook-Tabelle und vollständiges Label-Schema stehen in [Grafana Alloy](./alloy.md).

| Quelle | Methode | Labels |
| :--- | :--- | :--- |
| vm-nomad-server-04/05/06 | Ansible-Rolle `alloy` (systemd) | `source=journal` |
| vm-nomad-client-04/05/06 | Ansible-Rolle `alloy` (systemd) | `source=journal` |
| Container auf den Client-Nodes | Docker-Log-Treiber `journald`, gelesen von der Rolle | `source=journal` plus `nomad_job`, `nomad_task`, `nomad_group`, `nomad_namespace` |
| Vault Audit-Log (aktiver Server) | zweites Audit-Device Typ `syslog` ins Journal | `source=journal`, `app=vault-audit`, `signal=security` |
| vm-traefik-01/02 | Compose-Alloy (traefik-ha) | `source=docker-compose` |
| pve00, pve01, pve02 | Ansible-Rolle `alloy` (systemd) | `source=proxmox` |
| CheckMK | Ansible-Rolle `alloy` (systemd) | `source=checkmk` |
| PBS | Ansible-Rolle `alloy` (systemd) | `source=pbs` |
| Datacenter Manager | Ansible-Rolle `alloy` (systemd) | `source=datacenter-manager` |
| Logdateien (pveproxy, pve-firewall, CheckMK-Site, PBS, LINSTOR) | `local.file_match` plus `loki.source.file` | `app` je Datei, dazu `filename` |
| UDM-Pro, Access Points, Switches | Syslog RFC3164 an den Traefik-VIP | `job=syslog`, `host` |
| NAS HomeServer | Syslog RFC3164 an vm-nomad-client-06 | `job=syslog`, `host` |
| linstor-csi | Docker-Quelle des Nomad-System-Jobs | `nomad_task` |

## Log-Levels

| Komponente | Log-Level | Konfigurationsort |
| :--- | :--- | :--- |
| Loki | `warn` | `monitoring/loki.nomad` |
| Grafana | `info`, Logger `tsdb.loki` auf `critical` gefiltert | `monitoring/grafana.nomad` |
| Nomad | `INFO` | `ansible/roles/nomad/defaults/main.yml` |
| Consul | `WARN` | `ansible/roles/consul/defaults/main.yml` |
| Vault | `INFO` | `ansible/roles/vault/defaults/main.yml` |
| Authentik | `info` | `identity/authentik.nomad` |
| Traefik (Core) | `WARN` | `traefik.yml.j2` |
| Traefik (Access) | aktiv (JSON, stdout) | Filter: `statusCodes: 400-599` + `minDuration: 2s` + `retryAttempts`; Rotation via Docker-Log-Driver |

LogQL-Beispiele für die Loki-Datasource (uid: `loki-logs`) sind in [Grafana Alloy](./alloy.md#logql-beispiele) gepflegt.

## Verwandte Seiten

- [Monitoring Stack](./index.md) -- Übersicht, Architektur und Alert-Routing
- [Monitoring Betrieb](./betrieb.md) -- Grafana-Admin, Silencing, Backup-Monitoring, Wartung
- [Grafana Alloy](./alloy.md) -- Log-Collector, Deployment-Methoden und LogQL-Beispiele
