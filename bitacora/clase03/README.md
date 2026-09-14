# Bitácora Clase 3 — Telemetría: el dato que hace posible detectar

**Curso:** Implementación de un SOC con Herramientas Open Source · Capacitación USACH  
**Estudiante:** Danny Vega Pinilla  
**Fecha:** 14 de septiembre de 2026  
**Repo:** https://github.com/dannyvegapinilla-bot/soc-lab-vega  

El laboratorio corre en **Ubuntu 24.04 LTS** (`soc-dos` sobre Proxmox/KVM) con **Docker Engine nativo** (no Desktop ni WSL). Red `soc-net` (`172.28.0.0/16`) de la Clase 2.

Hoy dejé telemetría **en origen**: eventos que yo provoqué, con hora. Wazuh (Clase 4) todavía no entra. Entregable: dos víctimas + tabla de tres fuentes. Aporte al Capstone: 1 Linux + 1 Windows.

---

## Arquitectura de víctimas

- **Contenedor `victima-linux`:** endpoint secundario. Ahí fallé el SSH (`lab`, puerto `2222`) y quedaron `auth.log` / `lastb`. `auditd` **no** opera en Docker: el netlink de auditoría no está namespaced (`auditctl`: Operation not permitted). No usé `pid: host` ni `--privileged`.
- **VM `soc-dos`:** víctima Linux con kernel completo. Ahí `auditd` quedó `enabled 1` (pid distinto de 0), cinco reglas y el `useradd` de `intruso` / `ausearch`.
- **VM Windows `DESKTOP-VF92IFO`:** otra VM en Proxmox (cuenta `WindowsSOC`, 4 GB / 40 GB). Sysmon y auditoría nativa. **No** instalé Sysmon en el PC de estudio.

---

## Fase 1 — Víctima Linux con SSH y auditd

Levanté `victima-linux` con `docker compose up -d` en `~/soc-lab/compose/victima-linux` (`2222→22`). Entré con `docker exec -it victima-linux bash`.

Dentro instalé `openssh-server`, `auditd`, `sudo`, `vim`, `curl` y `rsyslog`. Los cinco primeros y el usuario `lab` (uid=1001) ya venían del **7-sep**, guardados en la imagen `victima-linux-lab:clase03`. El 14-sep `apt` dijo *already the newest version*. `rsyslog` no estaba: se instaló el 14-sep (de ese `apt` sí hay evidencia).

Arranqué `sshd` (`mkdir -p /run/sshd && /usr/sbin/sshd`) y `rsyslogd`. SSH no escribe `auth.log` solo: pasa por syslog. `imklog` no abre `/proc/kmsg` (normal en un contenedor); el `auth.log` de SSH igual se escribe.

`service auditd start` en el contenedor **falla**. `auditctl -l` responde Operation not permitted. El contenedor queda como endpoint SSH/syslog. La auditoría de verdad va en la VM.

En **`soc-dos`** instalé `auditd`, lo dejé con systemd y escribí las reglas con `sudo tee` y `>` (no `>>`):

| Regla | Para qué |
|--------|----------|
| `-w /etc/passwd -p wa -k identidad` | Cambios en la lista de usuarios |
| `-w /etc/shadow -p wa -k identidad` | Cambios en hashes de contraseña |
| `-w /etc/sudoers -p wa -k sudoers` | Quién puede ser root |
| `execve` b64 `euid=0` `auid>=1000` `-k exec_root` | Un usuario real ejecuta algo como root (64 bits) |
| Lo mismo en b32 | Sin esto hay un punto ciego en 32 bits |

`sudo auditctl -l` listó las cinco. `sudo auditctl -s` mostró **enabled 1** y **pid 52150**.

---

## Fase 2 — Generar eventos reales

**SSH (contenedor).** Desde `soc-dos` fallé varias veces:

`ssh -o StrictHostKeyChecking=no -o PreferredAuthentications=password -p 2222 lab@localhost`

Clave incorrecta. Simula fuerza bruta (T1110). Hora: host **14-sep-2026 14:14 UTC**; contenedor **11:14 −03**.

- `lastb`: `lab`, `ssh:notty`, origen **172.28.0.1** (gateway de `soc-net`).
- `auth.log`: `Failed password for lab from 172.28.0.1` a las **11:14:21 / 11:14:36 / 11:14:51 −03**. Las líneas del 7-sep las conservó el volumen.

`sudo aureport -au -i` en la VM salió *no events of interest*: los fallos de `lab` fueron al contenedor, no al SSH de `soc-dos`. Son **dos `auth.log` distintos**. No se mezclan.

**Auditd (VM).** El `echo` a `/etc/passwd` del PDF no deja rastro útil en Docker. Como `dvega` entré con `sudo -i` (no con `lab`). `lab` en la VM quedó uid=1001 y en `sudo`. Creé `intruso` (uid=1002) con `useradd -m -s /bin/bash intruso`.

`sudo ausearch -k identidad -i` muestra `exe=/usr/sbin/useradd`, `proctitle=useradd -m -s /bin/bash intruso`, escritura en `passwd`/`shadow`, `auid=dvega` y `euid=root`. Persistencia (T1136) con who-data.

Evidencia cruda: `bitacora/clase03/eventos-linux.txt`.

---

## Fase 3 — Víctima Windows con Sysmon

Otra VM en Proxmox, no mi Windows personal. PowerShell como administrador (`True` y `S-1-16-12288`).

En `C:\Tools` bajé `Sysmon.zip` y `sysmonconfig.xml` (SwiftOnSecurity) con `Invoke-WebRequest` (~19:43). El PDF asume que ya están.

Instalé con el comando del PDF: `Sysmon64.exe -accepteula -i`. **Sysmon 15.22**, config validada, servicio y driver en marcha. `Get-Service` = Running. A las **19:46 −03** (UtcTime 22:46) ya había eventos **ID 1**.

Encendí lo que Windows trae apagado (claves del PDF; `auditpol` con GUID porque está en español):

- `ProcessCreationIncludeCmdLine_Enabled` = `0x1` → 4688 con línea de comandos
- `EnableScriptBlockLogging` = `0x1` → 4104 en claro
- `{0CCE922B-…}` creación de procesos: Aciertos
- `{0CCE9215-…}` inicio de sesión: Aciertos y errores

No agrandé los logs a 512 MB: el disco iba al ~90 % (37 de 40 GB).

Provoqué a las **20:11 −03** (UtcTime 23:11): `whoami`, `net user intruso Lab.2026 /add`, `intruso` a Administradores, y `powershell.exe -EncodedCommand` (era `whoami` en Base64).

En origen:

- **Sysmon 1:** `whoami.exe`, hash, padre PowerShell con `-EncodedCommand`
- **4688:** mismo `whoami`, con línea de comandos; sin hash ni el Base64 del padre
- **4104:** el script ya dice `whoami`
- **4720:** alta de `intruso` (la hizo `WindowsSOC`) — paralelo del `useradd` en Linux

---

## Fase 4 — Tabla de normalización

Ver `tabla-normalizacion.md`. Un evento de cada fuente, campos comunes, y qué falta (sobre todo la IP y el huso horario).

---

## Preguntas de comprobación

1. **¿Por qué el 4688 sin línea de comandos tiene poco valor?**  
   Solo dice que se ejecutó un programa (por ejemplo `powershell.exe`), no **qué** se le pasó. Un `powershell -enc …` se ve igual que abrir PowerShell normal. Sin la clave del PDF no puedo responder en un incidente.

2. **¿Qué diferencia hay entre Sysmon 1 y el 4688?**  
   Los dos dicen “se creó un proceso”. Sysmon 1 trae hash, padre y la línea del padre (en mi lab el Base64). El 4688 es nativo, con menos campos; en el mío sí salió el comando de `whoami.exe` porque activé la clave. Se complementan: uno viene con Windows, el otro hay que instalarlo.

3. **¿Qué pasa si los relojes no están sincronizados?**  
   Mezclé −03 (contenedor y Windows local), UTC del host y UtcTime de Sysmon. Si cada máquina tiene su hora, un mismo ataque parece tres hechos distintos. Sin NTP no hay correlación.

4. **Dos fuentes que no ingeriría.**  
   Debug/verbose de aplicaciones: volumen enorme y casi no detecta ATT&CK. Conexiones de red de todos los endpoints si no hay capacidad: el PDF estima ~180 GB a 90 días frente a ~4 GB de autenticación. Primero auth, procesos con comando e integridad de archivos.

5. **¿Por qué una fuente que deja de reportar no alerta, y cómo se detecta?**  
   Deja de haber eventos, no aparece un “error”. El SIEM se queda callado. Se detecta con un catálogo (quién es dueño de la fuente) y un panel de salud: “esta máquina no mandó nada en N minutos”. La ausencia no significa que no haya amenaza.

---

## Fase 5 — Publicación

Commit y `git push` a `main` en el repo privado. En Moodle va la URL del repo; el docente como colaborador. Un Word solo en el PC no cuenta como entregado.
