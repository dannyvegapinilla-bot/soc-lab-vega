# Informe ejecutivo — Laboratorio 5

**Estudiante:** Danny Vega Pinilla
**Fecha:** 23 de septiembre de 2026
**Alcance:** solo lab (`victima-linux` 172.28.0.3 y agentes Wazuh en soc-dos). No hay exposición a internet.

## Resumen

En el lab el riesgo de red es bajo: GVM (Full and fast, con y sin SSH) no pasó de 2.1 y no abrió puertos. El volumen está en Wazuh: miles de CVE del kernel en victima-soc-dos. Ninguno de los CVE de esa captura está en CISA KEV y el EPSS de los dos que consulté es < 0,2 %. Priorizo actualizar linux-image, no el ICMP. Windows casi no aporta. Primero parchearía KEV o EPSS alto y expuesto; acá no se da.

## Hallazgos

| Activo | Hallazgo | CVSS | KEV | EPSS | Expuesto | Acción | Responsable | Plazo |
|--------|----------|------|-----|------|----------|--------|-------------|-------|
| victima-linux | ICMP Timestamp (GVM C6) | 2.1 Low | No | N/A | solo soc-net | registrar; no urgente | Danny Vega | lab |
| victima-linux | CPE Inventory (GVM auth) | Log | No | N/A | lab | ninguna | Danny Vega | — |
| victima-linux | traceroute (GVM auth) | Log | No | N/A | lab | ninguna | Danny Vega | — |
| victima-soc-dos | CVE-2026-89773 (linux-image 6.8.0-139) | 5.5 Medium (Red Hat) / 7.1 (Tenable) / "-" Wazuh | No | 0,18 % | no (local, lab, AMD) | actualizar kernel | Danny Vega | esta semana |
| victima-soc-dos | CVE-2026-89730 (mismo paquete) | 7.1 High (Tenable); NVD pendiente | No | 0,20 % | no (local, lab) | mismo apt de kernel | Danny Vega | esta semana |
| victima-soc-dos | CVE-2026-89693 (mismo paquete) | "-" / pendiente | No | no publicado | no | mismo parche | Danny Vega | esta semana |
| victima-soc-dos | CVE-2026-89627 (mismo paquete) | "-" / pendiente | No | no publicado | no | mismo parche | Danny Vega | esta semana |
| victima-soc-dos | Lote ~5103 CVE kernel | mixto / pendiente | No en esta muestra | bajo en los dos que medí | no | un upgrade de kernel | Danny Vega | esta semana |
| victima-win | 1 hallazgo (dashboard) | no abierto | no verificado | no verificado | LAN del lab | revisar Inventory Win | Danny Vega | lab |
| Cluster | GVM vs VD | GVM máx. 2.1; Wazuh miles de CVE | — | — | GVM ve red; Wazuh ve paquetes | usar las dos | Danny Vega | continuo |

Fuentes: mis reportes GVM C6, dashboard/inventory Wazuh, wazuh.com (CVE-2026-89773), Tenable/NVD, catálogo CISA KEV (consulta 23-sep-2026).
