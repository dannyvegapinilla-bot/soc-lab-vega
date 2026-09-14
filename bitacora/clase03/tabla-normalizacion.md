# Tabla de normalización — Clase 3

Estudiante: Danny Vega Pinilla
Fecha: 14 de septiembre de 2026

Tomé un evento de cada fuente, los que yo provoqué el 14-sep. No son el mismo hecho: en Linux fallé el SSH, en `soc-dos` creé `intruso` y en Windows lancé un `whoami` ofuscado. Los puse con los mismos nombres de campo para poder compararlos.

| Campo común | auth.log (contenedor) | auditd (VM soc-dos) | Windows / Sysmon |
|-------------|----------------------|---------------------|------------------|
| Marca temporal | 14-sep-2026 11:14:21 −03 (el host estaba en 14:14 UTC). También 11:14:36 y 11:14:51 | 14-sep-2026, cuando entré con `sudo -i` como `dvega` | 14-sep-2026 20:11:25 −03 (UtcTime 23:11:25) |
| Host | `victima-linux` | `soc-dos` | `DESKTOP-VF92IFO` |
| Usuario | `lab` | `auid=dvega` (corrió como `euid=root`) | `WindowsSOC` |
| IP de origen | `172.28.0.1` | No viene: lo hice sentado en la VM | No viene: lo hice sentado en la VM |
| Proceso / comando | `sshd`, clave incorrecta | `useradd -m -s /bin/bash intruso` | `whoami.exe`; lo lanzó PowerShell con `-EncodedCommand` |
| Resultado | Falló (`Failed password`) | Salió bien: quedó `intruso` | Salió bien; el alta de `intruso` en Windows es el 4720 |

## Qué campo falta

Lo que más se echa de menos es la IP. En el `auth.log` sí está (`172.28.0.1`). En auditd y en Sysmon de este lab no, porque no fue un login por red. Si después quiero unir “quién” y “desde dónde”, SSH me da la IP y las otras dos me dan el comando y el usuario de verdad. Sin campos comunes parecen tres relatos distintos. Por eso hay que normalizar. Las horas tampoco hablan igual: el contenedor en −03, el host en UTC y Windows con hora local más UtcTime. Si los relojes no coinciden, no puedo correlacionar.
