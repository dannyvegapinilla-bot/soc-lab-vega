# Clase 2 — Docker: base del laboratorio SOC

## Plataforma
Ubuntu 24.04 en Proxmox (soc-dos). Docker Engine nativo. Red `soc-net` 172.28.0.0/16.

## Fase 1 — Comandos (contenedor demo nginx)

| Comando | Para qué | Qué vi |
|---------|----------|--------|
| `docker run -d --name demo -p 8080:80 nginx:alpine` | Crear y dejar el proceso en segundo plano | Contenedor Up, 8080->80 |
| `docker ps` | Qué está corriendo | `demo` con nginx:alpine |
| `docker logs demo` | Salida del PID 1 (diagnóstico) | nginx ready; en Clase 4 aquí se ve el indexador |
| `docker exec -it demo sh -c "ls /usr/share/nginx/html"` | Entrar al namespace de archivos | `index.html` 50x.html |
| `docker stats --no-stream` | CPU/RAM reales | ~8 MiB; sin limits del Compose |
| `docker inspect demo \| grep -i ipaddress` | Metadatos / IP | 172.17.0.2 (bridge por defecto) |
| `docker rm -f demo` | Destruir el contenedor | `ps -a` vacío; sin volumen se pierde todo |

`up` reconcilia el YAML y recrea si cambió. `start` solo reanuda contenedores ya creados. Tras editar Compose siempre `up`.

## Fase 2 — Red y persistencia

- `alpine-b` hizo ping a `alpine-a` por **nombre** (resolvió 172.28.0.2). Las IP cambian al recrear; por eso en Clase 5 el agente usará `wazuh.manager`.
- Escribí `persiste` en `/datos` (volumen `lab-datos`), borré `alpine-a` y un contenedor nuevo leyó el mismo texto. El dato vive en el volumen, no en el contenedor.

## Fase 3 — Compose victima-linux

`compose/victima-linux/docker-compose.yml` levantó con `ubuntu:24.04`, `sleep infinity`, puerto 2222.

Salida de `docker compose ps`:
- victima-linux Up, 0.0.0.0:2222->22/tcp

Salida de `docker stats`:
- MEM USAGE / LIMIT: 2.242MiB / 1GiB  (el tope de 1g está activo)

## Hardening (qué riesgo evita cada opción)

| Opción | Riesgo que evita |
|--------|------------------|
| `ubuntu:24.04` (no latest) | Que mañana baje otra versión y el lab deje de ser reproducible (15 % de la nota) |
| `cap_drop: ALL` + `cap_add` puntual | Que un compromiso use capacidades de red/dispositivos/archivos del host. AUDIT_* es para auditd en Clase 3 |
| `no-new-privileges:true` | Que un proceso sin privilegios escale a root via setuid |
| `limits` 1 CPU / 1g | Que el contenedor coma toda la RAM de soc-dos (~10 GB en el host) |
| volumen `victima-logs` | Perder `/var/log` al recrear el servicio (eventos de Clase 3) |
| Nunca `--privileged` ni montar `docker.sock` | Entregar el host: el kernel es compartido |

## Preguntas de comprobación

1. `latest` muda sin aviso: no puedes explicar ni reproducir lo que corre.
2. `docker rm` sin volumen borra la capa de escritura: se pierde el dato.
3. Montar `docker.sock` es casi root en el host: el contenedor puede crear otros contenedores privilegiados.
4. `up` aplica el YAML; `start` no recrea si el descriptor cambió.
5. El DNS de `soc-net` resuelve el nombre del servicio; la IP no es estable.

## Preguntas de comprobación

1. `latest` muda sin aviso: no puedes explicar ni reproducir lo que corre.
2. `docker rm` sin volumen borra la capa de escritura: se pierde el dato.
3. Montar `docker.sock` es casi root en el host.
4. `up` aplica el YAML; `start` no recrea si el descriptor cambió.
5. El DNS de `soc-net` resuelve el nombre del servicio; la IP no es estable.
