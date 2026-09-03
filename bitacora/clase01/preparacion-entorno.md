# Guía 0 / Clase 1 — Preparación del entorno

## Plataforma
- Ubuntu 24.04 LTS en VM sobre Proxmox (KVM), host soc-dos.
- Por qué: Docker Engine nativo, sin WSL ni Docker Desktop. La víctima Windows de la Clase 3 será otra VM en el mismo Proxmox, no anidada.
- Riesgo aceptado: el host tiene ~10 GB RAM; el curso pide 12 GB mínimo.

## Verificación Docker
- docker 29.7.2 / docker compose v5.5.0 / git 2.43.0
- docker run --rm hello-world: OK
- vm.max_map_count = 262144
- Disco / : ~57 GB (LVM ampliado desde 29 GB)

## Imágenes
- wazuh/wazuh-manager:4.14.2
- wazuh/wazuh-indexer:4.14.2
- wazuh/wazuh-dashboard:4.14.2
- ubuntu:24.04

## Git / GitHub
- Repo privado: dannyvegapinilla-bot/soc-lab-vega
- SSH ed25519. Pública verifica, privada firma.
- Colaborador docente: pendiente a Clase 1.
