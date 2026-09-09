# Clase 2 — Docker, la base del laboratorio

Repo: https://github.com/dannyvegapinilla-bot/soc-lab-vega
Maquina: Ubuntu 24.04 en Proxmox (soc-dos).

La guia pide: tabla de comandos, salidas de ps y stats, prueba de persistencia y cada opcion de seguridad con el riesgo que mitiga.
Fotos: Fase-3.PNG (descriptor), Fase-3-1.PNG (up/ps/stats), Fase-4.PNG (config, volumen, red).

## Tabla de comandos

| Comando | Para que sirve | Que vi |
|---------|----------------|--------|
| docker run -d --name demo -p 8080:80 nginx:alpine | Crear contenedor en segundo plano y publicar puerto | demo Up, 8080->80 |
| docker ps | Listar lo que corre | nombre, imagen, estado, puertos |
| docker logs demo | Salida del PID 1; primera parada si falla | nginx ready; en Clase 4 el indexador |
| docker exec -it demo sh -c "ls /usr/share/nginx/html" | Entrar al namespace | index.html, 50x.html |
| docker stats --no-stream | CPU y RAM reales | demo ~8 MiB sin tope |
| docker inspect demo | Metadatos, IP, red | IP 172.17.0.2 |
| docker rm -f demo | Borrar aunque este Up | ps -a vacio: sin volumen se pierde todo |
| docker network create --subnet 172.28.0.0/16 soc-net | Red con DNS | Alpine se ven por nombre |
| docker volume create lab-datos | Disco que sobrevive al contenedor | ahi escribi persiste |
| docker compose up -d | Aplicar el descriptor | Container victima-linux Running |
| docker compose ps | Estado real | ver salida abajo |
| docker compose stats --no-stream victima-linux | Comprobar limite de RAM | ver salida abajo |
| docker compose config | Validar YAML | Fase-4.PNG |
| docker compose start | Solo reanuda lo ya creado | si edite el descriptor uso up |

## Salidas de ps y stats (Fase-3-1.PNG)

docker compose ps:
NAME            IMAGE                     COMMAND            STATUS         PORTS
victima-linux   victima-linux-lab:clase03 sleep infinity     Up 23 seconds  0.0.0.0:2222->22/tcp

docker compose stats --no-stream victima-linux:
CONTAINER ID   NAME            CPU %   MEM USAGE / LIMIT   MEM %
21712bcdae30   victima-linux   0.00%   408KiB / 1GiB       0.04%

El 1GiB esta aplicado. -no-stream (un guion) fallo; --no-stream salio bien.

## Prueba de persistencia

1. docker volume create lab-datos
2. alpine-a con -v lab-datos:/datos escribio persiste en /datos/prueba.txt
3. docker rm -f alpine-a
4. Un contenedor nuevo monto el mismo volumen: cat imprimio persiste

El dato vive en el volumen, no en el contenedor. Por eso /var/log va a victima-logs (Fase-4.PNG: victima-linux_victima-logs).

Red: alpine-b ping a alpine-a por NOMBRE (172.28.0.2). Las IP cambian; el agente usara wazuh.manager. Fase-4.PNG: victima-linux en soc-net.

## Cada opcion de seguridad y el riesgo concreto

El kernel lo comparte el host. Dar de mas es dar soc-dos. Se ve en Fase-3.PNG.

| Opcion | Riesgo concreto que mitiga |
|--------|----------------------------|
| Etiqueta fija ubuntu:24.04 / clase03, nunca latest | Que baje otra version y el lab deje de ser reproducible |
| sleep infinity | Que el contenedor salga con codigo 0 al nacer (sin PID 1 no hay servicio) |
| cap_drop ALL | Que un compromiso use capacidades de red, dispositivos o dueno de archivos del host |
| cap_add solo CHOWN SETUID SETGID AUDIT_* | Abrir todo por si acaso; AUDIT_* es para auditd en Clase 3 |
| no-new-privileges:true | Que un setuid escale a root dentro del contenedor |
| limits 1 CPU y memory 1g | Que se coma los 10 GB del Proxmox (stats: 408KiB / 1GiB) |
| Volumen victima-logs en /var/log | Perder logs al recrear (evidencia Clase 3) |
| soc-net external true | Que Compose invente otra red y no resuelva nombres |
| Puerto 2222:22 solo en el host | No publicar SSH de la victima a internet |
| Nunca --privileged ni docker.sock | Entregar el host: con el socket se crean contenedores privilegiados |

Nota Clase 3 (Fase-3.PNG): sume DAC_OVERRIDE, FOWNER, NET_BIND_SERVICE y SYS_CHROOT para apt y sshd. No es privileged.

## Trazabilidad
Repo privado, push a main. Sin push no esta entregado. gitignore bloquea .env y claves.
