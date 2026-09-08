# Limitación: auditd en contenedor

- Reglas escritas en /etc/audit/rules.d/soc.rules (passwd, shadow, sudoers, exec_root).
- auditd arranca; auditctl -l / ausearch: Operation not permitted.
- Causa: el kernel/host no expone netlink de audit al contenedor (ni con AUDIT_* ni seccomp:unconfined).
- No se usó --privileged (regla del curso).
- Evidencia Linux de esta clase: auth.log + lastb (autenticación SSH).
- Integridad de archivos (auditd) se reintentará en host Linux o se cubre con Sysmon en Windows.
