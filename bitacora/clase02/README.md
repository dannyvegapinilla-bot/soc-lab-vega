# Bitácora Clase 2 — Docker: la base sobre la que se construye el SOC

**Curso:** Implementación de un SOC con Herramientas Open Source · Capacitación USACH  
**Estudiante:** Danny Vega Pinilla  
**Repo:** https://github.com/dannyvegapinilla-bot/soc-lab-vega  

El entorno operativo es **Ubuntu 24.04** en Proxmox (`soc-dos`). No uso Docker Desktop: el motor es **Docker Engine nativo**.

Capturas en esta carpeta: `Fase-3.PNG`, `Fase-3-1.PNG`, `Fase-4.PNG`.

---

## Verificación previa

Antes del laboratorio comprobé que Docker ya venía de la Guía 0 / Clase 1: `docker --version`, `docker compose version`, `docker run --rm hello-world` y `vm.max_map_count = 262144`. Sin eso la Clase 2 no arranca.

---

## Fase 1 — Comandos esenciales

Practiqué el ciclo de vida con un nginx liviano (`demo`) para no llegar en blanco al Compose.

| Comando | Para qué sirve | Qué vi |
|---------|----------------|--------|
| `docker run -d --name demo -p 8080:80 nginx:alpine` | Crear el contenedor en segundo plano y publicar un puerto | `demo` Up, `8080->80` |
| `docker ps` | Ver qué está corriendo | Nombre, imagen, estado, puertos |
| `docker logs demo` | Salida del proceso principal (PID 1). Primera parada si algo falla | nginx listo; en Clase 4 aquí se mira el indexador |
| `docker exec -it demo sh -c "ls /usr/share/nginx/html"` | Entrar al namespace y listar archivos | `index.html`, `50x.html` |
| `docker stats --no-stream` | CPU y RAM reales | ~8 MiB, **sin** tope de Compose |
| `docker inspect demo` | IP, red, volúmenes | `172.17.0.2` (bridge por defecto) |
| `docker rm -f demo` | Eliminar aunque esté en ejecución | `ps -a` vacío |

Al borrar `demo` no quedó nada: **sin volumen, los archivos se van con el contenedor**.

---

## Fase 2 — Red y persistencia

Creé `soc-net` (`172.28.0.0/16`) y el volumen `lab-datos`.

**Red.** Dos Alpine en esa red. Desde `alpine-b` hice ping a **`alpine-a` por nombre** (resolvió `172.28.0.2`), no por IP. Las direcciones cambian al recrear el contenedor;  En `Fase-4.PNG` se ve `victima-linux` unida a `soc-net`.

**Persistencia (paso a paso):**

1. `docker volume create lab-datos`
2. `alpine-a` montó `-v lab-datos:/datos` y escribió `persiste` en `/datos/prueba.txt`
3. `docker rm -f alpine-a`
4. Un contenedor **nuevo** montó el mismo volumen: `cat` imprimió **`persiste`**

El dato no estaba “dentro de alpine-a”: estaba en el volumen. Por eso `/var/log` de la víctima va a `victima-logs` (`Fase-4.PNG`: `victima-linux_victima-logs`).



## Fase 3 — Primer Compose con hardening

El descriptor está en compose/victima-linux/. En Fase-3.PNG se ve abierto. Cada opción, con el riesgo concreto que mitiga:

| Qué hay en el Compose | Qué hace | Riesgo concreto si no está |
|------------------------|----------|----------------------------|
| image: ubuntu:24.04 (después imagen victima-linux-lab:clase03) | Fija la versión; no uso latest | Que mañana baje otra cosa y el lab deje de ser reproducible |
| command: sleep infinity | El PID 1 sigue vivo | El contenedor nace y se apaga (código 0) |
| networks: soc-net | La mete en la red de la Fase 2 | Los servicios no se ven por nombre |
| ports: 2222:22 | El 2222 del host es el 22 del contenedor | SSH de la víctima expuesto o inaccesible para el lab |
| cap_drop ALL + cap_add puntual | Quita el kernel de más y devuelve solo CHOWN, SETUID, SETGID, AUDIT_* | Un compromiso usa red cruda, discos o dueño de archivos del host |
| no-new-privileges:true | Impide ganar privilegios nuevos | Un setuid escala a root dentro del contenedor |
| limits: 1 CPU, memory: 1g | Cota cgroups | Se come los ~10 GB del Proxmox y se congela la clase |
| volumes: victima-logs:/var/log | Los logs sobreviven al borrar el contenedor | Se pierde la evidencia de la Clase 3 |
| soc-net: external: true | Reutiliza la red ya creada; Compose no inventa otra | El agente no resuelve nombres |
| Nunca --privileged ni docker.sock | No entregar el demonio | Control del host desde el contenedor |

docker compose up -d levantó el servicio. Evidencia en Fase-3-1.PNG.

---

## Verificación final — salidas de ps y stats

De Fase-3-1.PNG:

    docker compose ps
    NAME            IMAGE                     COMMAND            STATUS         PORTS
    victima-linux   victima-linux-lab:clase03 "sleep infinity"   Up             0.0.0.0:2222->22/tcp

    docker compose stats --no-stream victima-linux
    CONTAINER ID   NAME            CPU %   MEM USAGE / LIMIT   MEM %
    21712bcdae30   victima-linux   0.00%   408KiB / 1GiB       0.04%

El 1GiB está aplicado (no es un comentario). En Fase-4.PNG: compose config válido, volumen victima-linux_victima-logs, contenedor en soc-net.

---

## Preguntas de comprobación

**1. ¿Por qué usar latest es un riesgo en un entorno de seguridad?**
Porque latest no es una versión: es lo último que haya hoy. Puede cambiar sin aviso, dejar de ser compatible con el compose y, en un incidente, no poder explicar qué artefacto estaba corriendo. Una etiqueta fija (4.14.2, 24.04) se replica igual en cualquier equipo.

**2. ¿Qué se pierde al ejecutar docker rm si no se usó un volumen?**
Todo lo que se escribió en la capa del contenedor: archivos, configs, logs. Es almacenamiento temporal. El rm se lo lleva. Lo demostré: sin volumen, demo desapareció con su contenido; con lab-datos, persiste siguió ahí.

**3. ¿Por qué montar /var/run/docker.sock dentro de un contenedor es peligroso?**
Ese socket es el demonio Docker del host. Quien lo tiene puede crear contenedores, montar el disco del anfitrión y, en la práctica, ser root de soc-dos. No es un archivo más: es el plano de control.

**4. ¿Qué diferencia hay entre docker compose up y docker compose start?**
up mira el descriptor y deja el estado real igual al YAML (crea, recrea si cambió). start solo enciende contenedores que ya existían; si edité el archivo, start no se entera. Después de tocar el Compose siempre up.

**5. ¿Por qué el agente de la Clase 5 apuntará a un nombre y no a una IP?**
Porque la IP del contenedor cambia al recrearlo (alpine-a resolvió 172.28.0.2 esa vez; la siguiente puede ser otra). El DNS de soc-net resuelve el nombre del servicio (wazuh.manager) y la conexión no se rompe.

---
