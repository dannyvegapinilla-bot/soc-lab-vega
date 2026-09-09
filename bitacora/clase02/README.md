# Clase 2 — Docker, la base del laboratorio

Repo: https://github.com/dannyvegapinilla-bot/soc-lab-vega
Maquina: Ubuntu 24.04 en Proxmox (soc-dos), Docker Engine nativo.
Entregable: compose/victima-linux/docker-compose.yml

Esta bitacora sigue la rubrica (100 puntos). Escribi lo que hice y por que, no una copia del PDF.
Capturas en esta carpeta: Fase-3.PNG, Fase-3-1.PNG, Fase-4.PNG.

## 1. Compose (40 pts)

El YAML esta en compose/victima-linux/docker-compose.yml.
En otro equipo: crear la red soc-net (external: true, Compose no la crea) y docker compose up -d.

En soc-dos levanto con ubuntu:24.04 (nunca latest), sleep infinity, puerto 2222:22, limite 1 CPU y 1 GB, volumen victima-logs en /var/log.

Lo que vi:
victima-linux  ubuntu:24.04  sleep infinity  Up  0.0.0.0:2222->22/tcp
MEM USAGE / LIMIT: 2.242MiB / 1GiB

Ese 1GiB lo aplica Docker de verdad, no es un comentario. Antes probe nginx:alpine (demo) con run, ps, logs, exec, stats, inspect y rm. Al borrarlo, ps -a quedo vacio: sin volumen se pierde todo.

up reconcilia el YAML; start solo despierta lo ya creado. Si edito el archivo, siempre up.

## 2. Red y volumen (20 pts)

Red soc-net 172.28.0.0/16: alpine-b hizo ping a alpine-a por NOMBRE (resolvio 172.28.0.2). Las IP cambian al recrear; por eso el agente de la Clase 5 usara wazuh.manager.

Volumen lab-datos: escribi persiste en /datos, borre alpine-a y un contenedor nuevo leyo el mismo texto. El dato vive en el volumen. Por eso /var/log de la victima va a victima-logs.

## 3. Fundamentacion (20 pts)

El kernel lo comparte el host. Si le doy de mas al contenedor, le doy soc-dos.

- ubuntu:24.04: que latest no cambie el lab de un dia para otro.
- cap_drop ALL y cap_add puntual: que un compromiso no use red/discos/dueno de archivos del host. AUDIT_* es para auditd.
- no-new-privileges: que un setuid no salte a root dentro del contenedor.
- limits 1 CPU / 1g: que no se coma los 10 GB del Proxmox.
- volumen en /var/log: no perder eventos al recrear.
- Nunca --privileged ni docker.sock: el socket es casi root en el host.

Nota Clase 3: con solo las caps del PDF, apt y sshd fallaron. Sume DAC_OVERRIDE, FOWNER, NET_BIND_SERVICE y SYS_CHROOT, comentado en el YAML. No es privileged.

## 4. Trazabilidad (20 pts)

Repo privado soc-lab-vega, push a main. Mensajes utiles:
- Clase 2: Compose de la victima Linux con limites y hardening
- Clase 2: completar preguntas de comprobacion en bitacora

gitignore bloquea .env, claves y certificados. Si no hice push, no esta entregado.

## Preguntas al cerrar

1. latest muda sin aviso; no puedo defender lo que bajo Docker Hub esa manana.
2. docker rm sin volumen borra la capa de escritura.
3. docker.sock permite levantar contenedores privilegiados y salir al host.
4. up aplica el YAML; start no se entera si lo edite.
5. Nombre, no IP: la de hoy no es la de manana.

## Verificacion

cd ~/soc-lab/compose/victima-linux
docker compose ps
docker compose config
docker stats --no-stream victima-linux
docker volume ls | grep victima
docker network inspect soc-net
