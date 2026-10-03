# Orquestación de Contenedores y Volúmenes (Podman)

## Objetivo

La infraestructura utiliza **Podman en modo Rootless** para ejecutar los servidores de Minecraft de forma aislada respecto al sistema operativo host.

La configuración de los servicios se define mediante archivos compatibles con **Compose**, utilizando `podman-compose` como herramienta de despliegue.

El objetivo de esta capa es separar:

- La configuración de los servicios.
- Los datos persistentes de Minecraft.
- El sistema operativo host.
- Los procesos individuales de cada servidor.

Esta separación permite administrar los servidores de forma reproducible sin otorgar privilegios de `root` a los procesos de Minecraft.

---

## 1. Modelo de ejecución

Los servidores de Minecraft se ejecutan dentro de contenedores independientes.

```text
┌─────────────────────────────────────────────┐
│ Arch Linux — Bare Metal                     │
│                                             │
│ ┌─────────────────────────────────────────┐ │
│ │ Podman Rootless                         │ │
│ │                                         │ │
│ │ ┌──────────────┐  ┌──────────────┐     │ │
│ │ │ Minecraft    │  │ Minecraft    │     │ │
│ │ │ Modpack 1    │  │ Modpack 2    │ ... │ │
│ │ └──────┬───────┘  └──────┬───────┘     │ │
│ │        │                  │             │ │
│ └────────┼──────────────────┼─────────────┘ │
│          │                  │               │
│     Persistent Data    Persistent Data      │
└──────────┼──────────────────┼───────────────┘
           │                  │
           ▼                  ▼
      ./server/data      ./server/data
```

Cada servidor dispone de su propio espacio de datos y ciclo de vida.

Un fallo o reinicio de un contenedor no implica necesariamente la modificación del sistema operativo host ni de los demás servicios.

---

## 2. Infraestructura como código

El repositorio contiene la configuración necesaria para reproducir la infraestructura de los servicios.

Los archivos Compose definen aspectos como:

- Imágenes utilizadas.
- Variables de entorno.
- Puertos publicados.
- Montajes de almacenamiento.
- Nombre de los servicios.
- Dependencias entre contenedores.
- Configuración necesaria para iniciar cada servidor.

Los datos generados durante la ejecución **no forman parte de la infraestructura como código**.

Esto permite mantener el repositorio ligero y evitar introducir información mutable o potencialmente sensible en el control de versiones.

---

## 3. Gestión de datos y volúmenes

Cada servidor mantiene sus datos persistentes dentro de un directorio independiente:

```text
./<nombre-servidor>/data
```

Por ejemplo:

```text
minecraft/
├── java-modpack/
│   ├── compose.yml
│   └── data/
│
├── java-vanilla/
│   ├── compose.yml
│   └── data/
│
└── bedrock/
    ├── compose.yml
    └── data/
```

Dentro de estos directorios pueden encontrarse:

- Mundos.
- Mods.
- Configuraciones.
- Logs.
- Archivos generados por el servidor.
- Información persistente de los contenedores.

Estos datos permanecen fuera de la definición principal de la infraestructura y pueden respaldarse independientemente.

---

## 4. Separación entre configuración y datos

La arquitectura distingue explícitamente entre **configuración declarativa** y **estado persistente**.

```text
Repositorio Git
│
├── compose.yml
├── .env.example
├── .gitignore
└── documentación
        │
        ▼
   Configuración
        │
        ▼
     Podman
        │
        ▼
   ┌───────────────┐
   │ Minecraft     │
   │ Container     │
   └───────┬───────┘
           │
           ▼
      ./data/
           │
           ├── Mundos
           ├── Mods
           ├── Configuración
           └── Logs
```

El repositorio describe **cómo debe ejecutarse el servicio**, mientras que `data/` contiene **el estado generado por el servicio**.

Esta separación facilita tanto el control de versiones como las estrategias de respaldo y recuperación.

---

## 5. Permisos y etiquetado SELinux

Cuando el entorno utiliza un sistema con SELinux habilitado, los montajes de volumen pueden utilizar el modificador `:Z`:

```yaml
volumes:
  - ./java-modpack/data:/data:Z
```

El modificador `:Z` permite aplicar el etiquetado SELinux adecuado al contenido montado para su utilización por el contenedor.

Esto resulta especialmente relevante en entornos donde SELinux aplica controles adicionales sobre el acceso de los procesos a los archivos.

> **Nota:** `:Z` está relacionado con el etiquetado de seguridad de SELinux. El aislamiento proporcionado por Podman Rootless es una característica independiente del modelo de ejecución sin privilegios.

---

## 6. Control de versiones

Los datos persistentes de los servidores no deben almacenarse directamente en Git.

Los directorios `data/` se excluyen mediante `.gitignore`:

```gitignore
**/data/
```

De esta manera, el repositorio conserva únicamente los archivos necesarios para reconstruir la configuración de los servicios.

No deben incluirse en Git:

- Mundos de Minecraft.
- Logs.
- Archivos temporales.
- Credenciales.
- Tokens.
- Claves privadas.
- Archivos `.env` con valores reales.
- Datos generados por los servidores.

Cuando sea necesario documentar variables de entorno, se debe proporcionar una plantilla:

```text
.env.example
```

sin incluir secretos reales.

---

## 7. Despliegue en Arch Linux Bare Metal

Los contenedores se ejecutan directamente sobre el nodo Arch Linux mediante Podman Rootless.

Los servicios que requieren exposición de red se vinculan exclusivamente a la dirección privada de WireGuard:

```yaml
services:
  java-modpack:
    image: itzg/minecraft-server

    ports:
      - "10.10.0.2:${PUERTO_TCP}:25565/tcp"

    volumes:
      - ./java-modpack/data:/data:Z
```

Este diseño evita utilizar un bind genérico como:

```yaml
ports:
  - "${PUERTO_TCP}:25565"
```

cuando el servicio debe permanecer accesible únicamente a través de la interfaz de red privada.

La dirección:

```text
10.10.0.2
```

corresponde al nodo Arch Linux dentro de la red WireGuard.

Por tanto, el flujo esperado es:

```text
Internet
   │
   ▼
Oracle Linux
   │
   │ Firewall / Forwarding
   ▼
WireGuard
   │
   ▼
10.10.0.2
   │
   ▼
Podman
   │
   ▼
Minecraft
```

---

## 8. Principio de aislamiento

La utilización de Podman Rootless establece una frontera adicional entre los procesos de Minecraft y el sistema operativo host.

El objetivo no es asumir que el contenedor es una barrera de seguridad absoluta, sino reducir los privilegios disponibles para los procesos y limitar el impacto potencial de un fallo dentro del servicio.

La arquitectura combina varias capas:

```text
┌─────────────────────────────────────┐
│ Internet                            │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Oracle Linux                        │
│ Firewall / Gateway                 │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ WireGuard                           │
│ Red privada                         │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Arch Linux                          │
│                                     │
│ Podman Rootless                     │
│        │                            │
│        ├── Minecraft Modpack 1      │
│        ├── Minecraft Modpack 2      │
│        └── Minecraft Vanilla        │
│                                     │
└─────────────────────────────────────┘
```

Cada capa cumple una función diferente:

**Firewall → Segmentación → Aislamiento → Servicio**

Este modelo evita depender de una única medida de seguridad y proporciona una base sobre la cual pueden añadirse posteriormente controles adicionales, como límites de recursos, políticas de red, monitorización, backups y unidades administradas mediante `systemd` o Quadlet.
