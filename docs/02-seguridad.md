k# Seguridad Base y Firewall

Estrategia **Zero Trust** implementada en ambos hosts mediante herramientas nativas, puertos ofuscados y monitoreo continuo de logs.

## 1. Firewalls y Listas de Control de Acceso

### Arch Linux (nftables)
Implementación limpia usando el reemplazo moderno de iptables.
* **Política:** `Drop` por defecto en la cadena `input` y `forward`.
* **Reglas de Acceso:** Se permite SSH en un puerto no estándar y todo el tráfico proveniente explícitamente de la interfaz virtual `wg0` (`iifname "wg0" accept`).

### Oracle Linux (firewalld & OCI Security Lists)
* **OCI Cloud:** Puertos `1024-1029` y `51820/UDP` (WireGuard) habilitados en las Security Lists.
* **Firewalld:** Zona `trusted` asignada a la interfaz `wg0`. Se utiliza enmascaramiento (`masquerade`) y reenvío de puertos directo para canalizar tráfico hacia Arch Linux minimizando la latencia.

## 2. Prevención de Intrusiones (Fail2ban)

### Oracle Linux (EPEL 10)
Debido a la arquitectura Enterprise Linux 10, la instalación se gestiona mediante el repositorio Extra Packages for Enterprise Linux (EPEL).
* **Paquetes:** `fail2ban`, `fail2ban-firewalld`, `fail2ban-systemd`.
* **Configuración:** Modular a través de `/etc/fail2ban/jail.d/sshd.local`.
* **Reglas:** 5 intentos fallidos en 10 minutos provocan un bloqueo (`banaction = firewallcmd-rich-rules`) de 24 horas sobre el puerto SSH ofuscado.

### Arch Linux
Uso nativo de `fail2ban` leyendo directamente el journal de `systemd` para bloquear intentos de intrusión desde la red local o a través del túnel VPN.

## 3. Auditoría de Kernel (Auditd)
Demonio `auditd` configurado en ambos servidores para registrar a nivel de kernel cualquier modificación en los archivos de autenticación (`/etc/passwd`, `/etc/shadow`) y configuraciones de red (`sshd_config`, `firewalld`).
