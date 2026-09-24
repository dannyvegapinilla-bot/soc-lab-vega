# Bitácora Clase 6 — Laboratorio 5

**Curso:** Implementación de un SOC con Herramientas Open Source · Capacitación USACH
**Estudiante:** Danny Vega Pinilla
**Fecha:** 23 de septiembre de 2026
**Repo:** https://github.com/dannyvegapinilla-bot/soc-lab-vega

Levanté GVM (Greenbone) y usé el módulo Vulnerability Detection de Wazuh 4.14.2 en `soc-dos` (Ubuntu 24.04, Proxmox). Escaneé solo `victima-linux` en `soc-net`. Las claves quedaron en `.env`, no aquí.

## Arquitectura

| Pieza | Dónde | Rol |
|-------|--------|-----|
| GVM `immauss/openvas:26.07.27.01-slim` | Docker, `:9392` | Escaneo activo (Full and fast) |
| `victima-linux` | `172.28.0.3` / `soc-net` | Único target autorizado |
| Wazuh VD | dashboard `:8443` | Inventario pasivo (paquetes → CVE) |
| Agentes | `victima-soc-dos`, `victima-win` | Fuente del inventario |

Compose publicado: `compose/gvm/docker-compose.yml`. El `.env` no va a Git.

## Fase 1 — Levantar GVM

El tag de la guía (`22.5.05`) dio 404. Probé `26.07.27.01`: el restore llenó el disco (55/57 GB). Borré el volumen y pasé al slim (tope 4 GB, `soc-net`, 9392). El slim se caía al actualizar SCAP; dejé `SKIPSYNC=true`, quité el SCAP a medias y creé `admin` con `gvmd`. Sin Feed Import Owner no había Scan Configs; con el UUID de admin aparecieron las 7, incluida Full and fast (~191557 NVT). Panel en `http://192.168.1.206:9392`.

## Fase 2 — Sin credenciales

Tarea **C6 sin credenciales** contra `172.28.0.3`, Full and fast, sin SSH. Done (~2 min): 6 results, 1 CVE, máxima **2.1 Low** (ICMP Timestamp, QoD 80). Desde la red, sin usuario, casi no hay superficie.

## Fase 3 — Con credenciales

`lab` ya existía; `sshd` no corría. Lo arranqué y creé `lab-ssh` + target `victima-linux-auth`. Tarea **C6 con credenciales** (el nombre en GVM quedó NameC6 por un pegado). Done: **14** results (2 Low ICMP + 12 Log: CPE Inventory, traceroute). Tope igual 2.1. Más filas = mejor visibilidad, no “está peor”.

## Fase 4 — Wazuh VD

`vulnerability-detection` ya venía `enabled=yes` (60 min). No toqué `ossec.conf`. Dashboard: `victima-soc-dos` ~5102 hallazgos (Ubuntu 24.04, `linux-image-6.8.0-139`); `victima-win` 1. Inventory: 5103 hits, severidad “-”. Abrí CVE-2026-89773 en wazuh.com: Red Hat **5.5 Medium**, vector local (drm/amd/display).

## Fase 5

Informe en `informe-vulnerabilidades.md`. Prioricé actualizar el kernel de `soc-dos`, no el ICMP. Ningún CVE de la muestra está en CISA KEV; EPSS de los dos que medí < 0,2 %.

## Qué no va a Git

`.env`, `wazuh-docker/` y claves. El Word es anexo. Moodle = este repo.
