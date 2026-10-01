# Seguridad Base y Firewall

Esta sección documenta la capa inicial de seguridad para el host físico y el bastión en la nube. Se ha priorizado el uso de autenticación asimétrica (llaves SSH) y puertos no estándar para mitigar el escaneo automatizado. Los puertos exactos se mantienen fuera del control de versiones por seguridad.

## Políticas de SSH
Ambos servidores tienen deshabilitada la autenticación por contraseña (`PasswordAuthentication no`) y el acceso directo de root (`PermitRootLogin prohibit-password` o `no`).

* **Arch Linux (Bare Metal):** Puerto `<SSH_PORT_ARCH>`
* **Oracle Linux (Bastión):** Puerto `<SSH_PORT_ORACLE>`

## Reglas de Firewall

### Arch Linux (nftables)
Se utiliza `nftables` configurado con una política de denegación por defecto (`policy drop`) en la cadena de entrada.
* **Puertos Abiertos:** `<SSH_PORT_ARCH>/tcp` (SSH), ICMP (Ping).
* **Tráfico permitido:** Loopback (`lo`) y estado de conexiones activas (`established, related`).

### Oracle Linux (firewalld & Cloud Security Lists)
Manejo de reglas mediante `firewalld` a nivel de SO y Security Lists a nivel de Oracle Cloud Infrastructure (OCI).
* **Zona Activa:** Public (por defecto).
* **Puertos Abiertos SO:** `<SSH_PORT_ORACLE>/tcp` (SSH). El servicio `ssh` estándar ha sido removido.
* **SELinux:** Configurado para aceptar el puerto `<SSH_PORT_ORACLE>` bajo el contexto `ssh_port_t`.
* **OCI Security List:** Ingress rule configurada para permitir tráfico TCP al puerto `<SSH_PORT_ORACLE>`.

## Prevención de Intrusiones y Auditoría (Fail2ban & Auditd)

Se implementa una arquitectura **Zero Trust** donde ambos nodos ejecutan monitoreo continuo de logs y llamadas al sistema.

### Fail2ban
* **Backend:** `systemd` (Journald) en ambos hosts.
* **Arch Linux:** Umbral de 3 reintentos fallidos en un lapso de 10 minutos. Ban temporal de 2 horas.
* **Oracle Linux:** Política agresiva para red pública. Umbral de 3 reintentos fallidos, provocando un Ban de 24 horas en `firewalld`.

### Auditd (Kernel Auditing)
Se han establecido reglas de auditoría para rastrear intentos de modificación en archivos de sistema críticos:
* Cambios en `/etc/ssh/sshd_config` (Regla: `sshd_config`).
* Alteraciones en cuentas de usuario y contraseñas (`/etc/passwd`, `/etc/shadow`).
* Modificaciones de reglas en `firewalld` (Exclusivo en Oracle Linux).

### Oracle Linux (Fail2ban & EPEL 10)
Debido a la naturaleza de Enterprise Linux 10, la instalación requiere el repositorio EPEL:
* **Paquetes:** `epel-release`, `fail2ban`, `fail2ban-firewalld`, `fail2ban-systemd`.
* **Configuración modular:** `/etc/fail2ban/jail.d/sshd.local`.
* **Acción de bloqueo:** `firewallcmd-rich-rules` (Integración nativa con `firewalld`).
* **Regla:** 5 reintentos fallidos en 10 minutos generan un ban de 24 horas en `firewalld`.
