# Bitácora Clase 5 — Laboratorio 4

**Curso:** Implementación de un SOC con Herramientas Open Source · Capacitación USACH  
**Estudiante:** Danny Vega Pinilla  
**Fecha:** 22 de septiembre de 2026  
**Repo:** https://github.com/dannyvegapinilla-bot/soc-lab-vega  

Enrolé agentes Wazuh **4.14.2** sobre el stack de la Clase 4 (`soc-dos`, Proxmox). Manager + agente Linux en la misma VM; víctima Windows = otra VM (`DESKTOP-VF92IFO`, `192.168.1.207`). Las claves no van aquí: `.env` y `wazuh-docker/` están fuera de Git.

## Arquitectura

| Quién | Dónde | Cómo habla |
|-------|--------|------------|
| Manager (Docker) | soc-dos | 1515 enrolar, 1514 eventos, dashboard `:8443` |
| Agente 001 `victima-soc-dos` | misma VM | `127.0.0.1`, grupo `linux-victimas` |
| Agente 002 `victima-win` | Windows 11 | `192.168.1.206`, grupo `windows-victimas` |

No enrolé el contenedor `victima-linux`.

## Fases 1–2

`wazuh-authd` y `wazuh-remoted` en running. IP LAN `192.168.1.206` (`ens18`). Desde Windows, `Test-NetConnection` a 1514 y 1515 = True. Recreé los cuatro errores de la guía (netstat en la víctima, `IP_MANAGER`, ping `-c` vs `-n`, ICMP bloqueado hacia Windows). Wazuh no usa ping.

## Fases 3–5

Grupos **antes** de instalar: `linux-victimas` y `windows-victimas`. Agente Linux 4.14.2-1 (no 4.14.7), GPG en dos pasos, `apt-mark hold`. Windows: MSI 4.14.2-1; el servicio arrancó `Stopped` porque corrí `NET START` demasiado pronto; luego `Running` y `Connected to the server`. `agent_control -l`: 001 y 002 **Active**, `default` en cero.

## Fases 6–7

Shared file hash **antes:** 001 `2ebcd862…` · 002 `f7502018…`.  
**Después** de `agent.conf`: 001 `ef89357c044880031c6b91b1d5afa960` · 002 `8824d79fa4e17797531e53e3c4dfb82e`.  
Linux: `audit.log` en `merged.mg`. Windows: Sysmon/Operational, PowerShell y Security filtrado (4624, 4625, 4688, 4720…). Configs en este directorio (`agent-linux.conf`, `agent-win.conf`).

## Fase 8

Linux: SSH fallido ×3 → regla **2502** nivel 10 (el PDF dice 5712; anoté la mía). `useradd` / FIM / `chmod` de `/etc/shadow` (777 y vuelta a 640).  
Windows: `net user intruso-lab` → evento **4720**, regla **60109** nivel 8, ATT&CK **T1098**. PowerShell `-EncodedCommand` → 4688 + Sysmon 1, regla **92057** nivel 12 (T1059).  
Volumen en `alerts.json`: Linux **438**, Windows **3234**.

El dashboard estuvo vacío: Filebeat hacía **401** porque el manager tenía `SecretPassword` y el indexer ya no. Puse `${INDEXER_PASSWORD}` en el Compose, recreé el manager y Threat Hunting mostró las alertas (**549** en 2 h). Búsquedas: `data.win.system.eventID: 4720` (1 hit, `victima-win`) y `rule.id: 2502` (2 hits, `victima-soc-dos`).

## Qué no va a Git

`.env`, certificados y el árbol `wazuh-docker/` (Compose con secretos). El Word es anexo; Moodle = este repo.
