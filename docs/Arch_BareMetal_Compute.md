# Arch Linux Bare Metal: nodo de cómputo

## Propósito

Este nodo aloja los contenedores de Minecraft, procesa los mundos y proporciona los recursos de memoria y almacenamiento necesarios para la operación de los servidores.

## Especificaciones del equipo

| Recurso | Valor |
|---|---|
| Procesador | Intel Xeon E5-2667 v3 |
| Memoria RAM | 16 GB |
| Almacenamiento | NVMe de 1 TB |
| Sistema operativo | Arch Linux |
| Dirección IP de WireGuard | `10.10.0.2` |

> **Importante:** La dirección `10.10.0.2` debe utilizarse únicamente dentro de la red privada de WireGuard. No debe publicarse directamente en Internet.

---

## 1. Persistencia de las sesiones de usuario

Los contenedores se ejecutan mediante **Podman en modo rootless**, es decir, sin privilegios de superusuario. En este modo, los servicios pertenecen al usuario que los ejecuta.

Para que los servicios continúen activos después de cerrar una sesión SSH, es necesario habilitar `linger` para el usuario correspondiente.

Ejecuta el siguiente comando con el usuario que administra los contenedores:

```bash
loginctl enable-linger "$USER"
```

Puedes verificar que `linger` está habilitado con:

```bash
loginctl show-user "$USER" -p Linger
```

La salida esperada es:

```text
Linger=yes
```

Esta configuración permite que el administrador de servicios del usuario continúe funcionando aunque no exista una sesión SSH activa.

### Reinicio automático de contenedores

Si el entorno utiliza una unidad de usuario para reiniciar contenedores de Podman, puede habilitarse con:

```bash
systemctl --user enable --now podman-restart.service
```

Antes de ejecutar este comando, confirma que la unidad exista:

```bash
systemctl --user list-unit-files | grep podman
```

> **Nota:** Si `podman-restart.service` no está disponible, debe utilizarse el mecanismo de reinicio definido por el proyecto, por ejemplo, unidades generadas con **Quadlet** o archivos de servicio de `systemd`.

Para comprobar el estado de los servicios del usuario:

```bash
systemctl --user --type=service
```

---

## 2. Exposición de puertos

Los servidores de Minecraft no deben exponerse directamente a Internet desde este nodo.

La regla principal es:

> Los puertos internos de los servidores deben estar disponibles únicamente a través de la interfaz o dirección de WireGuard.

No deben publicarse directamente los puertos estándar:

- `25565/tcp`: Minecraft Java Edition.
- `19132/udp`: Minecraft Bedrock Edition.

En lugar de exponerlos públicamente, los contenedores pueden vincularse a la dirección privada de WireGuard:

```yaml
services:
  dawncraft:
    ports:
      - "10.10.0.2:1024:25565/tcp"

  divine-journey:
    ports:
      - "10.10.0.2:1025:25565/tcp"

  bedrock:
    ports:
      - "10.10.0.2:1028:19132/udp"
```

### Mapeo de puertos

| Servicio | Dirección accesible | Puerto interno |
|---|---|---|
| DawnCraft | `10.10.0.2:1024` | `25565/tcp` |
| Divine Journey | `10.10.0.2:1025` | `25565/tcp` |
| Bedrock | `10.10.0.2:1028` | `19132/udp` |

Estos puertos deben permanecer restringidos a la red privada. El acceso público debe gestionarse desde el nodo que actúe como **proxy o punto de entrada**.

### Recomendaciones adicionales

- No utilizar `0.0.0.0` en los mapeos de puertos si el servicio solo debe ser accesible mediante WireGuard.
- No abrir los puertos `1024`, `1025` ni `1028` en el firewall público.
- Permitir únicamente el tráfico procedente de las direcciones autorizadas de WireGuard.
- Mantener separados los puertos de administración y los puertos destinados a los jugadores.
- No incluir contraseñas, tokens ni claves privadas en los archivos Compose.
- Utilizar variables de entorno o mecanismos de gestión de secretos cuando sea necesario.

---

## 3. Comandos de diagnóstico

### Comprobar puertos TCP

Para verificar los puertos TCP asociados a la dirección de WireGuard:

```bash
ss -tulnp | grep '10.10.0.2'
```

También puedes comprobar específicamente los puertos de Minecraft Java:

```bash
ss -ltnp | grep -E '1024|1025'
```

### Comprobar el puerto UDP de Bedrock

Para verificar el puerto UDP utilizado por Bedrock:

```bash
ss -lunp | grep '10.10.0.2'
```

O de forma específica:

```bash
ss -lunp | grep '1028'
```

### Revisar los contenedores activos

```bash
podman ps
```

Para consultar también los contenedores detenidos:

```bash
podman ps -a
```

### Consultar los registros de un contenedor

Para consultar los últimos 100 registros:

```bash
podman logs --tail 100 NOMBRE_DEL_CONTENEDOR
```

Para seguir los registros en tiempo real:

```bash
podman logs -f NOMBRE_DEL_CONTENEDOR
```

### Verificar el estado de los servicios del usuario

```bash
systemctl --user --failed
```

Si existe un servicio específico para el despliegue:

```bash
systemctl --user status NOMBRE_DEL_SERVICIO
```

---

## 4. Consideraciones de seguridad

Este nodo debe considerarse un **servidor interno de la infraestructura**.

Como mínimo, deben aplicarse las siguientes medidas:

- Mantener el sistema operativo actualizado.
- Utilizar autenticación SSH mediante claves en lugar de contraseñas.
- Deshabilitar el acceso SSH directo del usuario `root`.
- Restringir SSH mediante firewall y, preferiblemente, mediante WireGuard.
- Ejecutar los contenedores con Podman rootless.
- Evitar almacenar secretos en el repositorio Git.
- Utilizar archivos `.env.example` para documentar las variables necesarias sin incluir sus valores reales.
- Realizar copias de seguridad de los mundos y de las configuraciones.
- Supervisar el uso de memoria, CPU, almacenamiento y red.
- Revisar periódicamente los puertos que permanecen a la escucha.

### Comprobar los puertos abiertos localmente

```bash
ss -tulpen
```

### Revisar el espacio disponible

```bash
df -h
```

### Consultar el uso de memoria

```bash
free -h
```
