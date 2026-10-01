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
