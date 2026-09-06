---
title: Grafana Alloy
description: Log-Collector für alle Infrastruktur-Komponenten -- Sammelwege, Label-Schema und Troubleshooting
tags:
  - service
  - monitoring
  - logging
  - alloy
---

# Grafana Alloy

Grafana Alloy ist der zentrale Log-Collector im Homelab. Er liest auf jedem Host das systemd-Journal und ausgewählte Logdateien, empfängt Syslog von Netzwerkgeräten und schiebt alles nach Loki. Container-Logs kommen seit dem 06.09.2026 nicht mehr über den Docker-Socket, sondern über das Journal.

## Übersicht

| Attribut | Wert |
| :--- | :--- |
| Deployment | Ansible-Rolle `alloy` (systemd) auf allen Hosts, Compose-Container auf den Traefik-VMs, Nomad-System-Job `system/alloy.nomad` als Rest-Sammler |
| Version | über APT-Pin und `dpkg-hold` eingefroren, Hebung nur über die Rollen-Variable |
| Ziel | `loki.service.consul:3100` |
| Lesepositionen | persistent je Sammelweg |
| Secrets | keine |

## Rolle im Stack

Alloy ist die einzige Schicht, die Logzeilen einsammelt. Fällt sie auf einem Host aus, wird dieser Host in Loki still, ohne dass ein Alert der Log-Regeln das anzeigt. Genau diese Lücke schliesst der periodische Job `loki-deadman`, der ausserhalb von Grafana prüft und an Uptime Kuma pusht.

## Architektur

**Leitfrage:** Wie kommt eine Logzeile von ihrer Quelle nach Loki, und wer merkt es, wenn keine mehr kommt?

Lese-Konvention: Der Pfeil zeigt vom Initiator zum Ziel. Durchgezogene Kanten sind synchrone Abruf- oder Schreibflüsse, gestrichelte Kanten sind ereignisgetriebene oder verbindungslose Ströme. Blau ist der Sammelweg, grün die Auswertung, orange die Selbstüberwachung.

```d2
classes: {
  svc: { style: { border-radius: 8 } }
  agent: { style: { border-radius: 8; stroke-dash: 2 } }
  box: { style: { border-radius: 8; stroke-dash: 4 } }
  db: { shape: cylinder; style: { border-radius: 8 } }
  ingest: { style: { stroke: "#1a73e8" } }
  push: { style: { stroke: "#1a73e8"; stroke-dash: 3 } }
  query: { style: { stroke: "#188038" } }
  event: { style: { stroke: "#188038"; stroke-dash: 3 } }
  watch: { style: { stroke: "#e8710a" } }
  watchpush: { style: { stroke: "#e8710a"; stroke-dash: 3 } }
}

quellen: Quellen {
  class: box

  container: Nomad-Container {
    class: svc
    tooltip: Alle Container der Client-Nodes, Log-Treiber journald
  }
  vault: Vault {
    class: svc
    tooltip: Audit-Device vom Typ syslog, nur der aktive Node schreibt
  }
  journal: systemd-Journal je Host { class: db }
  dateien: Logdateien {
    class: svc
    tooltip: pveproxy, pve-firewall, CheckMK-Site, LINSTOR, PBS
  }
  netz: NAS und UniFi { class: svc }
  compose: Compose-Container der Traefik-VMs { class: svc }
}

alloy: Grafana Alloy {
  class: box

  rolle: Ansible-Rolle als systemd-Dienst { class: agent }
  vip: Compose-Alloy am Traefik-VIP { class: agent }
  rest: Nomad-System-Job als Rest-Sammler { class: agent }
}

loki: Loki { class: db }
grafana: Grafana { class: svc }
keep: Keep { class: svc }
kuma: Uptime Kuma { class: svc }
jobs: loki-deadman und loki-rule-lint { class: agent }

quellen.container -> quellen.journal: 1. Log-Treiber journald { class: push }
quellen.vault -> quellen.journal: 2. Audit über syslog { class: push }
alloy.rolle -> quellen.journal: 3. liest Cursor { class: ingest }
alloy.rolle -> quellen.dateien: 4. tailt Dateien { class: ingest }
alloy.vip -> quellen.compose: 5. liest Docker-Socket { class: ingest }
quellen.netz -> alloy.vip: 6. Syslog 1514 am VIP { class: push }
alloy.rest -> loki: 7. Rest-Zeilen (HTTP) { class: push }
alloy.rolle -> loki: 8. pusht Zeilen (HTTP) { class: push }
alloy.vip -> loki: 8. pusht Zeilen (HTTP) { class: push }
grafana -> loki: 9. fragt ab (LogQL) { class: query }
grafana -> keep: 10. Webhook bei Regel-Verletzung { class: event }
jobs -> loki: 11. prüft Ingest und Selektoren { class: watch }
jobs -> kuma: 12. pusht Heartbeat { class: watchpush }
kuma -> keep: 13. Webhook bei Down { class: event }
```

Lesehilfe:

1. Der Docker-Daemon der Client-Nodes schreibt jede Container-Zeile über den [journald-Treiber](#container-logs-uber-den-journald-treiber) ins Journal des Hosts.
2. Vault schreibt sein Audit-Log über ein [zweites Audit-Device vom Typ syslog](#vault-audit-log) in dasselbe Journal.
3. Der Alloy-Dienst der [Ansible-Rolle](#journal-sammlung-durch-die-ansible-rolle) liest das Journal mit persistentem Cursor.
4. Derselbe Dienst tailt die konfigurierten [Datei-Targets](#datei-targets).
5. Auf den Traefik-VMs liest ein Compose-Alloy die Container des dortigen Docker-Compose-Stacks.
6. NAS, UniFi-Gateway, Switches und Access Points senden ihr [Syslog](#syslog-empfang) an den Keepalived-VIP.
7. Der [Nomad-System-Job](#nomad-system-job-als-rest-sammler) sammelt nur noch, was der Journal-Weg nicht erreicht.
8. Alle Alloy-Instanzen pushen an `loki.service.consul:3100`.
9. Grafana wertet die [Log-Regeln](./referenz.md#log-basierte-alert-rules-loki) mit LogQL aus.
10. Verletzte Regeln gehen als Webhook an [Keep](./keep/).
11. Die beiden periodischen Jobs prüfen den Ingest und die Regel-Selektoren unabhängig von Grafana.
12. Sie pushen ihr Ergebnis an Uptime Kuma statt an Keep, damit die Prüfung nicht an derselben Kette hängt wie das, was sie prüft.
13. Bleibt der Push aus, meldet Kuma Down an Keep.

**Belegt gegen** `ansible/roles/alloy/templates/config.alloy.j2`, `system/alloy.nomad`, `standalone-stacks/traefik-ha/templates/alloy-config.alloy.j2` und die Loki-Streams des Homelab-Clusters, Stand 06.09.2026.

## Container-Logs über den journald-Treiber

Die Nomad-Client-Konfiguration setzt für alle Container den Docker-Log-Treiber `journald` und hängt über `extra_labels` die Nomad-Labels an. Damit steht jede Container-Zeile im Journal des Nodes, auch die eines Zwei-Sekunden-Batch-Containers, den die frühere Docker-Discovery nie zu sehen bekam. Konfiguration in `ansible/roles/nomad/templates/client.hcl.j2` hinter der Variablen `nomad_docker_journald_logging`.

**Warum dieser Weg:** Die Docker-Discovery sah nur, was zum Scan-Zeitpunkt lief, und brauchte dafür den Docker-Socket im Container. Der Journal-Weg erfasst kurzlebige Tasks lückenlos, kommt ohne Socket aus und beendet zugleich die frühere Doppelerfassung des Journals durch System-Job und Ansible-Rolle.

**Wie die Labels ankommen:** Der Treiber übernimmt die Container-Labels ohne Präfix als Journal-Felder, aus `com.hashicorp.nomad.job_name` wird also das Feld `COM_HASHICORP_NOMAD_JOB_NAME`. Alloy bildet daraus die Labels `nomad_job`, `nomad_task`, `nomad_group` und `nomad_namespace`.

::: warning extra_labels und Treiber gehören in denselben Client-Neustart
Ein `extra_labels` ohne Treiberwechsel würde die Logs aller noch nicht rotierten Services still verlieren: die Drop-Regel des System-Jobs kann "Label plus json-file" nicht von "Label plus journald" unterscheiden. Umgekehrt entstünde ohne `extra_labels` nie ein `nomad_job`-Label.
:::

Bewusst nicht gesetzt ist die Treiber-Option `tag`. Sie würde `CONTAINER_TAG` und `SYSLOG_IDENTIFIER` auf denselben Wert setzen, und der Nomad-Containername enthält die Alloc-ID -- das erzeugte je Allocation einen neuen `syslog_identifier` und damit genau die Kardinalität, die das Label-Schema abbaut.

Die Labels `unit`, `syslog_identifier` und `level` setzt Alloy nur für Nicht-Container-Zeilen. Bei Container-Zeilen wäre `unit` immer `docker.service`, `syslog_identifier` trüge die kurze Container-ID, und der Treiber stuft stderr pauschal auf Priorität `err` ein.

Laufende Container behalten ihren Treiber bis zur nächsten Rotation. Alle 66 Service-Jobs wurden am 06.09.2026 in zwei Wellen rotiert, die periodischen Jobs ziehen beim nächsten Lauf von selbst nach.

::: warning Bekannter Verlustmodus
Ein Neustart von journald verwirft für einige Sekunden Container-Zeilen. Das ist der Preis dafür, dass kurzlebige Container überhaupt erfasst werden, und gegenüber der früheren Docker-Discovery eine Verbesserung.
:::

Damit die Container-Last das Journal nicht sprengt, setzt die Rolle auf den Clients ein journald-Drop-in mit 3 GB und sieben Tagen Aufbewahrung (abgeleitet aus der Grösse der Root-Partition) und verdoppelt das Rate-Limit. Für `docker.service` selbst schaltet `roles/nomad` das Rate-Limit ganz ab, weil dort alle Container-Zeilen zusammenlaufen.

## Journal-Sammlung durch die Ansible-Rolle

Der systemd-Dienst läuft auf allen Hosts ausser den Traefik-VMs. Die Konfiguration erzeugt Ansible aus `ansible/roles/alloy/templates/config.alloy.j2`.

| Playbook | Hosts | Source-Label | Zusätzlich |
| :--- | :--- | :--- | :--- |
| `deploy-alloy.yml` (Server-Play) | vm-nomad-server-04/05/06 | `journal` | Vault-Audit über das Journal |
| `deploy-alloy.yml` (Client-Play) | vm-nomad-client-04/05/06 | `journal` | Container-Zeilen, LINSTOR-Logs, vergrössertes Journal |
| `deploy-alloy-proxmox.yml` | pve00, pve01, pve02 | `proxmox` | pveproxy-Access-Log, Firewall-Log |
| `deploy-alloy-infra.yml` | CheckMK, PBS, Datacenter Manager | `checkmk`, `pbs`, `datacenter-manager` | Site- und Task-Logs |

Beide Plays in `deploy-alloy.yml` laufen mit `serial: 1` und prüfen nach jedem Host den Ready-Endpunkt, bevor der nächste an die Reihe kommt. Grund auf den Clients ist die Reboot-Disziplin der Storage-Nodes, Grund auf den Servern das Vault-Audit-Log: ein gleichzeitiger Restart liesse niemanden übrig, der in dem Moment liefert, und ein Config-Fehler fiele erst auf, wenn schon alle drei stehen.

Die Plays für `vm-vpn-dns-01` und den Zigbee-Node sind entfallen. Beide Hosts sind dekommissioniert, die Plays liefen bei jedem Aufruf ins Leere und verdeckten dabei echte Host-Fehler.

## Datei-Targets

Datei-Targets laufen über `local.file_match` und erst danach über `loki.source.file`. Der Grund ist eine Falle: `loki.source.file` expandiert Glob-Muster nicht selbst. Ein Pfad mit Stern bleibt dort ohne Treffer und ohne Fehlermeldung liegen, weshalb sechs von sieben Datei-Targets über Monate nie eine Zeile lieferten.

| Host | Datei | Label `app` |
| :--- | :--- | :--- |
| pve00, pve01, pve02 | `/var/log/pveproxy/access.log` | `pveproxy` |
| pve00, pve01, pve02 | `/var/log/pve-firewall.log` | `pve-firewall` |
| CheckMK | `cmc.log`, `web.log`, `notify.log` der Site `homelab` | `checkmk-core`, `checkmk-web`, `checkmk-notify` |
| PBS | `api/access.log` | `pbs-api` |
| PBS | `tasks/archive` | `pbs-tasks` |
| vm-nomad-client-04/05/06 | LINSTOR-Satellite-Logs | `linstor` |

Die Leserechte setzt die Rolle über POSIX-ACLs, nicht über Gruppenmitgliedschaften. `/var/log/pveproxy` ist `drwx------`, dort nützt die Mitgliedschaft in `www-data` ohne Durchsuchrecht nichts. Wo eine Datei ohne `create`-Direktive rotiert und der Dienst sie selbst neu anlegt, kommt eine Default-ACL auf das Verzeichnis dazu -- ohne sie wäre das Target nach der ersten Rotation wieder stumm. Das betrifft `cmc.log` der CheckMK-Site und `tasks/archive` auf dem PBS.

::: warning local.file_match listet jede Ebene eines Musters
Für die Glob-Auflösung braucht Alloy auf jedem Verzeichnis des Pfades Leserecht, Durchsuchrecht allein genügt nicht. Die CheckMK-Site liegt unter `drwxr-x--x`, ein Muster über `sites/*` lieferte deshalb eine leere Target-Liste, obwohl der direkte Lesetest als alloy-Benutzer funktionierte. Deshalb stehen dort feste Site-Pfade statt eines Globs.
:::

Beim PBS liest Alloy bewusst das Task-Archiv statt der Einzelprotokolle. Jede Task-Datei wäre wegen des `filename`-Labels ein eigener Loki-Stream, das sind 55'126 Dateien bei einem globalen Limit von 5'000 Streams. Das Archiv trägt eine Zeile je abgeschlossenem Task mit Endzeit und Status und ist damit ein einziger Stream und zugleich genau das, was ein Backup-Monitoring braucht.

::: info Stille Datei-Targets sind meist stille Quellen
Die LINSTOR-Ziele sind Fehlerberichte, die einmal geschrieben und nie wieder angefasst werden. Dass dort nichts nachkommt, ist der Normalzustand und sogar das gewünschte Signal.
:::

## Vault-Audit-Log

Vault schreibt sein Audit-Log über ein zweites Audit-Device vom Typ `syslog` mit Facility AUTH und Tag `vault` ins Journal des aktiven Nodes. Das Datei-Device bleibt als Rückfall aktiv, Alloy liest es nicht mehr.

**Warum nicht als Datei-Target:** Die Audit-Datei ist `0600 vault:vault` in einem `0750`-Verzeichnis. Alloy kommt nicht heran und meldet einen nicht lesbaren Pfad nirgends als Fehler -- die Lücke war dadurch über Monate vollständig stumm. Eine ACL hätte jede Rotation überleben müssen und wäre bei einem Fehler wieder stumm geblieben.

Ein rsyslog-Drop-in hält die Audit-Zeilen aus `/var/log/auth.log` heraus. Es bindet an Tag **und** Facility, nicht nur an den Tag: `vault.service` schreibt seine normalen Betriebsmeldungen unter demselben Programmnamen, aber mit Facility `daemon`. Ein reiner Tag-Filter hätte diese aus `/var/log/syslog` entfernt. Verwaltet in der Rolle `vault`, Rollout über `ansible/playbooks/deploy-vault-audit-syslog.yml`.

Dieselbe Unterscheidung braucht das Alloy-Relabel: Es setzt `app="vault-audit"` und `signal="security"` nur, wenn Identifier und Facility zusammen passen. Ohne die Facility-Bedingung wären auch die Betriebsmeldungen von `vault.service` als Audit und als Security-Signal ausgewiesen.

::: info Nur der aktive Node schreibt
Audit-Einträge entstehen ausschliesslich auf dem aktiven Vault-Node. Eine leere Audit-Datei auf einem Standby ist normal und kein Symptom. Ein Leader-Wechsel verschiebt die Quelle auf einen anderen `node`-Wert, weshalb der Deadman diese Quelle cluster-weit über das `app`-Label prüft und nicht je Node.
:::

## Syslog-Empfang

Der reguläre Empfänger ist der Compose-Alloy auf den Traefik-VMs, erreichbar über den Keepalived-VIP `10.0.2.20` auf Port 1514, TCP und UDP. UDM-Pro, Access Points und Switches senden dorthin.

**Format:** RFC3164 (BSD Syslog). UniFi und Synology senden dieses Format, nicht RFC5424.

**Label-Extraktion:** Das Feld `__syslog_message_hostname` wird auf das Label `host` gemappt, alle Zeilen tragen zusätzlich `job="syslog"`.

Das NAS HomeServer sendet seine SMB-Zugriffsprotokolle weiterhin an den Listener des Nomad-System-Jobs auf `vm-nomad-client-06`, nicht an den VIP. Deshalb hat der System-Job neben dem LINSTOR-CSI-Rest auch noch diesen Empfänger.

::: warning Testzeilen brauchen --rfc3164
`logger` sendet ohne diesen Schalter im Format RFC5424. Der Receiver läuft im rfc3164-Modus und verwirft solche Zeilen still, mit einem Parser-Fehler im Alloy-Log statt einer Zeile in Loki.
:::

## Nomad-System-Job als Rest-Sammler

Der System-Job `system/alloy.nomad` ist kein regulärer Sammelweg mehr. Er hält nur noch zwei Dinge:

- Die Container des System-Jobs `linstor-csi`. Er läuft privileged und hält die CSI-Volumes, deshalb ist er bewusst nicht rotiert und trägt weiterhin den json-file-Treiber. Seine Zeilen erreichen Loki nur über die Docker-Quelle dieses Jobs.
- Den Syslog-Port 1514 auf den Client-Nodes, solange das NAS HomeServer dorthin sendet.

Eine Drop-Regel auf das Label `com.hashicorp.nomad.job_name` hält alle Container mit journald-Treiber von der Docker-Quelle fern. Ohne sie läse der Job jede rotierte Allocation ein zweites Mal ein. Der Block `loki.source.journal` ist aus dem Job entfernt, das Journal der Clients liest ausschliesslich die Ansible-Rolle.

## Label-Schema

Die Labels sind so gewählt, dass sie eindeutige Identifikation erlauben und die Stream-Kardinalität in Loki begrenzen. Das globale Stream-Limit der Instanz liegt bei 5'000.

| Label | Wert (Beispiel) | Quelle | Gilt für |
| :--- | :--- | :--- | :--- |
| `node` | `vm-nomad-client-05` | `external_labels`, aus dem Hostnamen | alle Streams eines Collectors |
| `source` | `journal`, `proxmox`, `checkmk`, `pbs`, `datacenter-manager`, `docker-compose` | `external_labels`, je Playbook | alle Streams eines Collectors |
| `unit` | `consul.service` | Journal-Relabel | Nicht-Container-Zeilen |
| `syslog_identifier` | `vault`, `sshd` | Journal-Relabel | Nicht-Container-Zeilen |
| `level` | `err`, `warning` | Journal-Relabel aus `__journal_priority_keyword` | Nicht-Container-Zeilen |
| `nomad_job` | `browserless` | Journal-Feld des Log-Treibers | Container-Zeilen |
| `nomad_task` | `grafana` | Journal-Feld des Log-Treibers | Container-Zeilen |
| `nomad_group` | `browserless` | Journal-Feld des Log-Treibers | Container-Zeilen |
| `nomad_namespace` | `default` | Journal-Feld des Log-Treibers | Container-Zeilen |
| `app` | `pveproxy`, `pbs-tasks`, `vault-audit` | Datei-Target bzw. Relabel | Datei-Zeilen und Vault-Audit |
| `signal` | `security` | Relabel | Vault-Audit |
| `job` | `syslog` | statisches Label | Syslog-Empfang |
| `host` | `UDM-Pro-Lenzburg` | Syslog-Relabel | Syslog-Empfang |

**Structured Metadata statt Index-Label:** `container_id`, `container`, `image` und `nomad_alloc_id` sind abfragbar, erzeugen aber keine Streams. Sie sind die beiden Treiber der Kardinalität, die der Log-Treiber sonst in den Index schriebe. Ein Filter darauf funktioniert wie ein Label-Filter, nur zeigt `/series` sie nicht als Stream-Label.

**Warum `session-*.scope` gedroppt wird:** Diese systemd-Scopes entstehen bei jeder SSH-Verbindung und erzeugen hohe Kardinalität ohne diagnostischen Mehrwert.

::: info Zielschema hinter einem Schalter
Ein erweitertes Schema mit `cluster`, `role` und `source` als reiner Herkunftsart (`journal` gegen `file`) liegt in der Rolle hinter der Variablen `alloy_label_schema_v2` und ist ausgeschaltet. Es wird erst zusammen mit der Migration der Grafana-Regel-Selektoren scharf geschaltet, weil Dashboards und Regeln heute auf den Host-Klassen in `source` filtern.
:::

## Lesepositionen

Jeder Sammelweg legt seine Positionen auf ein persistentes Verzeichnis: die Rolle nach `/var/lib/alloy/data`, der System-Job über ein Host-Volume nach `/var/lib/alloy-nomad`, der Compose-Alloy nach `/var/lib/alloy-compose`. Ohne diese Persistenz las jeder Neustart das Journal bis zur konfigurierten Rückschau erneut ein, was am 15.08.2026 einen Duplikat-Burst von rund 234'000 Zeilen erzeugte.

::: warning Der Offset in positions.yml hinkt der Datei hinterher
Bei einem Datei-Target wird der gespeicherte Byte-Offset erst beim nächsten Lesevorgang nachgeführt. Bei einer Datei, die nach dem Erstlesen still liegt, steht er deshalb dauerhaft auf der Länge der ersten Zeile. Das ist reine Buchhaltung und keine Tail-Störung: `read_lines_total` zeigt die echte Zahl, und beim geordneten Herunterfahren schreibt Alloy die korrekte Endposition. Duplikate entstehen nur, wenn Alloy hart abgeschossen wird.
:::

## LogQL-Beispiele

Alle Abfragen in Grafana verwenden die Datasource **Loki** (uid `loki-logs`).

Container-Logs:

- `{nomad_job="grafana"}` -- alle Zeilen aller Tasks des Jobs
- `{nomad_task="loki"} |= "error"` -- Loki-Fehler
- `{nomad_job="browserless"} | container_id="46a435b4cff1"` -- Filter auf Structured Metadata

Host-übergreifend:

- `{node="vm-nomad-client-05"}` -- alle Zeilen von client-05
- `{source="proxmox"} |= "error"` -- Proxmox-Fehler
- `{unit="nomad.service"}` -- Nomad-Daemon-Logs

Dateien und Dienste:

- `{app="pveproxy"}` -- Proxmox-Web-Zugriffe
- `{app="pve-firewall"} |= "DROP"` -- verworfene Pakete
- `{app="pbs-tasks"} != " OK"` -- fehlgeschlagene Backup-Tasks
- `{app="checkmk-core"}` -- CheckMK-Core-Log
- `{app="vault-audit"}` -- Vault Audit-Trail
- `{signal="security"}` -- alle als sicherheitsrelevant gekennzeichneten Zeilen

Netzwerkgeräte:

- `{job="syslog"}` -- alle Syslog-Quellen
- `{job="syslog", host="UDM-Pro-Lenzburg"}` -- nur das Gateway

## Selbstüberwachung der Log-Pipeline

Zwei periodische Nomad-Jobs prüfen die Pipeline von aussen und melden über Uptime Kuma statt über Grafana und Keep. Der Grund ist Unabhängigkeit: Grafana wertet die Log-Regeln gegen dieselbe Loki-Instanz aus, deren Ausfall gemeldet werden soll, und Keep stuft Query-Fehler bewusst als Pipeline-Störung auf `warning` herab.

| Job | Takt | Prüft | Kuma-Monitor |
| :--- | :--- | :--- | :--- |
| `loki-deadman` | alle 5 Minuten | globaler Ingest plus jede erwartete Quelle | Loki-Deadman (Log-Ingest) |
| `loki-rule-lint` | täglich 06:20 | jeder Loki-Selektor der Alert-Regeln gegen sieben Tage Serien, dazu Abgleich der Kollektor-Labels gegen die Quellenliste | Loki Rule-Lint |

Der Deadman macht bis zu drei Durchgänge im Abstand von zwanzig Sekunden und pusht erst danach `down`. Zusammen mit den Kuma-Wiederholungen meldet ein andauernder Stall nach rund zwanzig Minuten, ein einzelner Fehlversuch gar nicht.

Die erwarteten Quellen stehen in `monitoring/loki-sources.txt` als eine Zeile je Quelle aus Label, Wert und Fenster. Beide Jobs binden dieselbe Datei ein, damit Prüfung und Abgleich nicht auseinanderdriften. Aufgenommen wird eine Quelle erst, wenn sie über mehrere Tage in jedem Fenster Zeilen hatte; ein einzelner Nachhol-Batch nach dem Erstanschluss ist kein Mass für die Dauerrate.

Der Rule-Lint überspringt Regeln mit der Annotation `lint: skip`. Sie ist für Regeln gedacht, deren Selektor legitim leer ist, weil er auf eine Oneshot-Unit zielt, die nur beim Booten schreibt.

::: warning Die Ausnahme ist eine Annotation, kein Label
Der Lint liest sie aus dem `annotations`-Block der Regel. Ein gleichnamiges Label bliebe wirkungslos, und die Regel würde weiterhin als stumm gemeldet.
:::

## Troubleshooting: Logs kommen nicht an

**1. Läuft Alloy?** Auf Rollen-Hosts `systemctl status alloy`, beim System-Job der Nomad-Job-Status. Der HTTP-Endpunkt `/-/ready` auf Port 12345 liefert 200, wenn Alloy bereit ist.

**2. Ist Loki erreichbar?** `loki.service.consul:3100` muss auflösbar sein. Alloy nutzt die Pi-hole-Resolver als DNS (Adressen: [Hosts und IPs](../_referenz/hosts-und-ips.md)).

**3. Sind die Targets aufgelöst?** Die Alloy-UI auf Port 12345 zeigt die Komponenten-Exports. Steht bei `local.file_match.files` eine leere Target-Liste, ist es ein Rechte- und kein Pfadproblem (siehe [Datei-Targets](#datei-targets)). Ein direkter Lesetest auf die Zieldatei als alloy-Benutzer beweist dabei nichts, weil ein Shell-Glob mit weniger Rechten auskommt.

**4. Fehlen Labels?** In Loki unter Explore prüfen, ob Zeilen mit dem erwarteten `node`-Label ankommen. Tragen Container-Zeilen kein `nomad_job`, ist entweder `extra_labels` nicht gesetzt oder der Container stammt noch aus der Zeit vor dem Client-Neustart und ist nicht rotiert.

**5. Kommt Syslog an?** Port 1514 auf dem empfangenden Host mit `ss` prüfen, TCP und UDP. Danach prüfen, ob das sendende Gerät auf den VIP zeigt.

**6. Loki antwortet mit HTTP 500 in der Nacht.** Das ist das erwartete Muster im vzdump-Fenster: Der Guest-Agent friert beim Start jedes VM-Backups das Dateisystem des Gasts ein, der laufende Push von Alloy läuft in seine Fünf-Sekunden-Frist und wird mit 500 quittiert. Ursache ist weder Storage-Bandbreite noch Loki selbst. Alloy überbrückt den Stall mit seinem Retry-Budget von rund achteinhalb Minuten, in 22 Tagen wurde deswegen keine einzige Charge verworfen. Kein Eingriff nötig.

## Verwandte Seiten

- [Monitoring Stack](./index.md) -- Gesamtübersicht mit den drei Pfaden und dem Loki-Backend
- [Monitoring Referenz](./referenz.md) -- Log-Quellen-Tabelle, Alert-Regeln und Log-Levels
- [Keep](./keep/) -- Incident-Hub, Signal-Klassifizierung und Pipeline-Störungen
- [Uptime Kuma](./uptime-kuma/) -- Push-Monitore der beiden Selbstüberwachungs-Jobs
- [Batch Jobs](../_querschnitt/batch-jobs.md) -- periodische Monitoring- und Wartungs-Jobs
- [Vault](../plattform/vault/index.md) -- Audit-Devices und Rollen-Konfiguration
