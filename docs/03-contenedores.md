# Orquestación de Contenedores y Volúmenes (Podman)

El entorno utiliza `podman-compose` en modo **Rootless**, aislando los procesos de Minecraft del sistema operativo host.

## Gestión de Datos y Volúmenes
La infraestructura como código (IaC) en este repositorio define la configuración de los servicios, pero ignora los datos pesados (mundos, mods).
* **Ruta de datos:** `./<nombre-servidor>/data`
* **Permisos:** Se utiliza el flag `:Z` en los montajes de volumen para garantizar compatibilidad estricta con sistemas de seguridad (SELinux / namespaces rootless).
* **Control de Versiones:** Todos los subdirectorios `/data/` están excluidos mediante `.gitignore`.

## Despliegue en Arch Linux (Bare Metal)
Los contenedores exponen sus servicios forzando el enlace (bind) exclusivo a la IP del túnel WireGuard (`10.10.0.2`), impidiendo accesos no autorizados desde otras interfaces físicas.

```yaml
# Ejemplo de servicio encapsulado
services:
  java-modpack:
    image: itzg/minecraft-server
    ports:
      - "10.10.0.2:${PUERTO_TCP}:25565/tcp"
    volumes:
      - ./java-modpack/data:/data:Z
