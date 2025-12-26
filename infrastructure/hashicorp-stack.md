---
title: Zigbee und Home Assistant
description:
published: true
date: 2025-12-26T17:45:50+00:00
tags:
editor: markdown
dateCreated: 2025-12-26T17:45:50+00:00
---

## Architektur

Zigbee Devices <-> Zigbee2MQTT (10.0.0.110) <-> Home Assistant (10.0.0.100)

## Services

| Service | IP | Port | Beschreibung |
|---------|-----|------|--------------|
| Zigbee2MQTT | 10.0.0.110 | 8080 | Web Interface |
| Mosquitto | 10.0.0.110 | 1883 | MQTT Broker |
| Portainer | 10.0.0.110 | 9000 | Docker Management |
| Home Assistant | 10.0.0.100 | 8123 | Smart Home |

## USB Adapter

- Silicon Labs CP210x (/dev/ttyUSB0)

## Befehle

    ssh sam@10.0.0.110
    cd docker
    docker-compose up -d
    docker logs zigbee2mqtt
