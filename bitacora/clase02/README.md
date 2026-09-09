# Clase 2 — Docker, la base del laboratorio

Repo: https://github.com/dannyvegapinilla-bot/soc-lab-vega
Maquina: Ubuntu 24.04 en Proxmox (soc-dos).
La evidencia son las fotos de esta carpeta, no un enlace a un yml.

| Foto | Que demuestra |
|------|----------------|
| Fase-3.PNG | Descriptor: imagen fija, caps, limites, volumen, red externa |
| Fase-3-1.PNG | El servicio levanta: up, ps, stats con tope 1GiB |
| Fase-4.PNG | YAML valido, volumen victima-linux_victima-logs y victima-linux en soc-net |

## 1. Compose (40 pts)

En otro PC: Docker, red soc-net (external: true) y docker compose up -d.

En Fase-3.PNG y Fase-3-1.PNG: imagen victima-linux-lab:clase03 (de ubuntu:24.04, nunca latest), sleep infinity, puerto 2222:22, 1 CPU / 1 GB, volumen en /var/log, cap_drop ALL y no-new-privileges.

Lo que vi en Fase-3-1.PNG:
Container victima-linux Running
0.0.0.0:2222->22/tcp
MEM USAGE / LIMIT: 408KiB / 1GiB

El 1GiB lo aplica Docker. El flag -no-stream (un guion) fallo; con --no-stream salio bien.

Antes: nginx demo. Al rm, ps -a vacio: sin volumen se pierde todo.
up reconcilia; start solo despierta. Si edito, siempre up.

## 2. Red y volumen (20 pts)

soc-net: alpine-b ping a alpine-a por NOMBRE (172.28.0.2). Las IP cambian; el agente usara wazuh.manager.
Fase-4.PNG: victima-linux en esa red.

Volumen lab-datos: escribi persiste, borre alpine-a, el dato siguio.
Fase-4.PNG: docker volume ls muestra victima-linux_victima-logs.

## 3. Fundamentacion (20 pts)

El kernel lo comparte el host. Lo de Fase-3.PNG:

- etiqueta fija: que latest no mueva el lab
- cap_drop ALL: que un compromiso no use red/discos/dueno del host
- no-new-privileges: que un setuid no salte a root
- limits 1g: que no se coma los 10 GB del Proxmox
- volumen /var/log: no perder eventos al recrear
- nunca privileged ni docker.sock: el socket es casi root

Nota Clase 3 (comentarios en Fase-3.PNG): sume DAC_OVERRIDE, FOWNER, NET_BIND_SERVICE y SYS_CHROOT para apt y sshd. No es privileged.

## 4. Trazabilidad (20 pts)

Repo privado, push a main.
- Clase 2: Compose de la victima Linux con limites y hardening
- Clase 2: bitacora alineada a la rubrica de evaluacion
gitignore bloquea secretos. Sin push no esta entregado.

## Preguntas
1. latest muda sin aviso.
2. rm sin volumen borra los datos.
3. docker.sock permite salir al host.
4. up aplica el descriptor; start no.
5. Nombre, no IP.
