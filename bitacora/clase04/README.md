# Bitácora Clase 4 — Laboratorio 3

**Curso:** Implementación de un SOC con Herramientas Open Source · Capacitación USACH  
**Estudiante:** Danny Vega Pinilla  
**Fecha:** 16 de septiembre de 2026  
**Repo:** https://github.com/dannyvegapinilla-bot/soc-lab-vega  

Wazuh 4.14.2 en Docker (`single-node`) sobre Ubuntu 24.04 (`soc-dos`, Proxmox). Dashboard en `https://192.168.1.206:8443`. Las contraseñas no se documentan: viven en `.env` (`.gitignore`).

---

## Verificación previa

`vm.max_map_count=262144`. RAM **15 Gi** (~14 Gi libres). Paré `victima-linux` para no pelear memoria con el indexer.

## Fase 1 — PKI

Cloné `wazuh-docker` con etiqueta **v4.14.2** (`detached HEAD` es normal). Generé certificados con `generate-indexer-certs.yml`. El `find: command not found` del generador no anuló la PKI. `ls` sin sudo dio Permission denied (archivos de root); con `sudo ls` vi CA, admin, indexer, manager y dashboard.

## Fase 2 — Compose

El YAML traía heap **1g** y el panel en **443**. Dejé 1g y puse **`8443:5601`**.

## Fase 3 — Despliegue

`docker compose up -d`: indexer, manager y dashboard Up. `curl` al `:9200` → **green**, 1 nodo. Yellow sería réplicas sin segundo nodo; red sería RAM/PKI. `wazuh-modulesd is running`. `wazuh-clusterd` no corre: un solo nodo. Certificado autofirmado en el navegador. Login de fábrica solo para comprobar; luego lo cambié.

## Fase 4 — Credenciales

Clave de fábrica = hallazgo. Generé hashes con `hash.sh` (JDK del contenedor) para **admin** y **kibanaserver**. Edité `internal_users.yml` (nano en Linux, no Notepad de Windows). La clave en claro está en `.env`; el Compose lee `${INDEXER_PASSWORD}` y `${DASHBOARD_PASSWORD}`. No cambié `API_PASSWORD`.

`securityadmin.sh` cargó los YAML al índice de seguridad (**Done with success**). Un `restart` del dashboard **no** aplica variables nuevas (el contenedor era de 38 h). Hizo falta recrear. `curl` con las cuentas nuevas: **green**. En capturas tapé las claves.

## Fase 5 — RBAC

Roles: **analista-n1** (lectura de agentes), **ing-deteccion** (eso más reglas/decoders), **admin-soc** (`security`/`cluster`/`agents` all). Usuario Wazuh **n1-vega** → `analista-n1`, *Allow run as* off.

El login del panel autentica contra el **indexer**, no contra el usuario de Security (API). Por eso di de alta `n1-vega` también en `internal_users.yml` (`kibanauser` + `readall`) y volví a correr `securityadmin`.

Como `n1-vega`, Index Management: `security_exception` (ni monitorizar índices). El analista de turno no debe poder borrar índices: si le roban la cuenta, no pueden borrar la evidencia.

## Fase 6 — Dimensionamiento

GB/día ≈ EPS × 86400 × bytes × (1 + réplicas) × 0,35 / 1 073 741 824

| | Laboratorio (200 EPS, 0 réplicas) | 500 endpoints (3000 EPS, 1 réplica) |
|--|-----------------------------------|-------------------------------------|
| GB/día | ~4 | ~118 |
| 90 días | ~360 GB | ~10,6 TB |

Con **1 TB**: en el lab caben ~256 días; yo dejaría **90**. En 500 endpoints el tera dura **~9 días**; 90 días no caben. Propondría 7 días en caliente o más disco. La retención se decide antes del incidente.

## Publicación

Este directorio `bitacora/clase04/` es lo que va a Git. **No** se versiona `.env` ni el árbol `wazuh-docker` con secretos.
