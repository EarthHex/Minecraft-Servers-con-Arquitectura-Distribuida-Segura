# DNS y Datos de Conexión

## Propósito

La infraestructura utiliza **DNS dinámico** para proporcionar un nombre de dominio estable que los jugadores puedan utilizar para conectarse a los servidores de Minecraft.

El dominio funciona como una capa de abstracción entre los jugadores y la dirección IP pública de la infraestructura. De esta manera, los jugadores no necesitan conocer directamente la dirección numérica de la instancia de Oracle Cloud.

**Proveedor DNS:** No-IP  
**Dominio:** `won-mc.servegame.com`

> **Nota:** DNS no constituye un mecanismo de seguridad ni oculta técnicamente la dirección IP pública frente a consultas DNS. Su función principal es proporcionar un nombre estable y facilitar la gestión de cambios de dirección.

---

## 1. Registro DNS

El dominio principal utiliza un registro de tipo `A` que apunta a la dirección IPv4 pública de la instancia de Oracle Cloud.

| Parámetro | Valor |
|---|---|
| Dominio | `won-mc.servegame.com` |
| Tipo de registro | `A` |
| Destino | IP pública de Oracle Cloud |
| Proveedor | No-IP |

El flujo general es:

```text
Jugador
   │
   │ won-mc.servegame.com
   ▼
DNS / No-IP
   │
   │ Resolución DNS
   ▼
IP pública de Oracle Cloud
   │
   ▼
Oracle Linux
```

Si la dirección IP pública cambia, únicamente debe actualizarse el registro DNS para que el dominio vuelva a apuntar al nuevo destino.

La infraestructura interna permanece independiente de este cambio porque la comunicación entre Oracle Linux y Arch Linux continúa utilizando la red privada de WireGuard.

```text
                    DNS
                     │
                     ▼
        won-mc.servegame.com
                     │
                     ▼
          IP pública de OCI
                     │
                     ▼
              Oracle Linux
                     │
              WireGuard VPN
                     │
                     ▼
               Arch Linux
```

---

## 2. Datos de conexión para jugadores

Los jugadores utilizan el mismo dominio como punto de entrada para los diferentes servidores.

La diferenciación entre servidores se realiza mediante los puertos públicos definidos en la arquitectura de red.

### Servidores Java

| Servidor | Dirección de conexión |
|---|---|
| DawnCraft | `won-mc.servegame.com:1024` |
| Divine Journey | `won-mc.servegame.com:1025` |
| Java adicional | `won-mc.servegame.com:1026` |

### Servidor Bedrock

| Servidor | Dirección | Puerto |
|---|---|---:|
| Bedrock Modpack | `won-mc.servegame.com` | `1028` |

En Minecraft Bedrock, el puerto debe especificarse cuando el cliente no utiliza el puerto predeterminado.

---

## 3. Flujo de conexión

Una conexión hacia uno de los servidores Java alojados en Arch Linux sigue aproximadamente este recorrido:

```text
Jugador
   │
   │ won-mc.servegame.com:1024
   ▼
DNS / No-IP
   │
   ▼
IP pública de Oracle Cloud
   │
   ▼
Oracle Linux
   │
   │ Firewall / Forwarding
   ▼
WireGuard
   │
   │ 10.10.0.2
   ▼
Arch Linux
   │
   │ Podman
   ▼
Minecraft Java
```

Para los servidores alojados directamente en Oracle Linux, el recorrido es más corto:

```text
Jugador
   │
   │ won-mc.servegame.com:1027
   ▼
DNS / No-IP
   │
   ▼
Oracle Linux
   │
   ▼
Minecraft Vanilla
```

De esta manera, DNS únicamente proporciona la resolución del nombre. El control posterior del tráfico corresponde a la capa de red y a los servicios que reciben la conexión.

---

## 4. Relación entre DNS, red y contenedores

La arquitectura separa claramente las responsabilidades de cada componente:

| Componente | Responsabilidad |
|---|---|
| No-IP / DNS | Resolver el dominio hacia la IP pública |
| Oracle Cloud | Proporcionar la infraestructura pública |
| Oracle Linux | Recibir y enrutar el tráfico |
| Firewall | Controlar qué tráfico puede acceder |
| WireGuard | Transportar tráfico hacia la red privada |
| Arch Linux | Ejecutar los servidores de Minecraft |
| Podman | Aislar y administrar los procesos |
| Minecraft | Gestionar las conexiones de los jugadores |

Esta separación evita que DNS tenga que conocer detalles internos de la infraestructura.

El dominio apunta únicamente al **punto de entrada público**, mientras que el direccionamiento interno permanece encapsulado dentro de la red privada.

---

## 5. Consideraciones de seguridad

El dominio DNS no debe considerarse una medida de ocultamiento o protección.

La seguridad se obtiene de las capas que existen después de la resolución DNS:

```text
             DNS
              │
              ▼
       Punto de entrada
              │
              ▼
          Firewall
              │
              ▼
        WireGuard VPN
              │
              ▼
       Nodo de cómputo
              │
              ▼
       Podman Rootless
              │
              ▼
      Servidor Minecraft
```

Por esta razón:

- El dominio no debe contener información sensible.
- Las credenciales nunca deben almacenarse en registros DNS ni en el repositorio.
- Los puertos innecesarios no deben exponerse únicamente porque exista un registro DNS.
- Las reglas de firewall deben mantenerse alineadas con los servicios realmente publicados.
- La dirección privada de WireGuard (`10.10.0.2`) no debe utilizarse como dirección pública de conexión.
- Los cambios de DNS deben comprobarse antes de modificar componentes internos de la infraestructura.

---

## 6. Principio de diseño

El DNS actúa como una **capa de direccionamiento**, no como una capa de seguridad.

El diseño busca que los jugadores solo necesiten conocer:

```text
won-mc.servegame.com
```

mientras que la infraestructura mantiene internamente la separación entre:

**Dominio → Entrada pública → Firewall → WireGuard → Nodo → Contenedor → Minecraft**

Esta abstracción permite modificar componentes internos de la infraestructura sin cambiar necesariamente la dirección que utilizan los jugadores.
