# Arquitectura de Red y Túnel Seguro (WireGuard)

## Objetivo

La infraestructura de Minecraft utiliza una arquitectura de red segmentada basada en **WireGuard**, con el objetivo de separar la exposición pública de los servidores de cómputo.

El diseño sigue una topología **Hub-and-Spoke**:

- **Oracle Linux** actúa como **Hub**, funcionando como punto de entrada público y gateway de la infraestructura.
- **Arch Linux** actúa como **Spoke**, alojando los servidores de Minecraft que requieren mayores recursos de cómputo.
- La comunicación entre ambos nodos se realiza exclusivamente mediante un túnel privado de **WireGuard**.

De esta manera, el nodo de cómputo permanece fuera de la exposición directa de Internet. Oracle Linux recibe las conexiones externas y únicamente reenvía el tráfico autorizado hacia los servicios correspondientes.

```text
                         INTERNET
                            │
                            │
                     ┌──────▼──────┐
                     │ Oracle Linux│
                     │    HUB      │
                     │ 10.10.0.1   │
                     └──────┬──────┘
                            │
                    WireGuard VPN
                    10.10.0.0/24
                            │
                     ┌──────▼──────┐
                     │  Arch Linux │
                     │   SPOKE     │
                     │ 10.10.0.2   │
                     └─────────────┘
                            │
                   ┌────────┴────────┐
                   │                 │
              Minecraft          Minecraft
              Modpacks            Modpacks
```

---

## 1. Topología de red privada

La comunicación entre los nodos se realiza mediante una red privada dedicada:

| Componente | Dirección / Configuración |
|---|---|
| Subred WireGuard | `10.10.0.0/24` |
| Oracle Linux — Hub | `10.10.0.1` |
| Arch Linux — Spoke | `10.10.0.2` |
| Keepalive del Spoke | `25s` |
| Tecnología VPN | WireGuard |

### Oracle Linux — Hub

Oracle Linux dispone de una dirección pública y constituye el único punto de entrada de Internet para los servicios que se encuentran detrás del túnel.

Su función principal es:

- Recibir conexiones externas.
- Aplicar las reglas de firewall.
- Determinar el destino del tráfico.
- Encaminarlas hacia los servicios correspondientes.
- Transportar el tráfico destinado a Arch Linux mediante WireGuard.

### Arch Linux — Spoke

Arch Linux funciona como nodo de cómputo interno.

Los servicios alojados en este nodo no requieren una dirección pública propia. El tráfico externo llega a través del túnel WireGuard y es entregado únicamente a los puertos previamente definidos.

Este aislamiento reduce la superficie de exposición del servidor de Minecraft y permite mantener separadas las funciones de **entrada de tráfico** y **procesamiento**.

---

## 2. Matriz de puertos y servicios

La infraestructura utiliza puertos externos independientes para diferenciar los distintos servidores de Minecraft.

> **Nota de seguridad:** utilizar puertos no convencionales puede reducir parte del ruido generado por escaneos automatizados, pero no constituye un mecanismo de seguridad por sí mismo. La protección efectiva depende principalmente del firewall, la segmentación de red, la autenticación y la correcta configuración de los servicios.

| Puerto público (OCI) | Protocolo | Servicio | Host físico | Acceso / Seguridad |
|---:|:---:|---|---|---|
| `1024` | TCP | Java Modpack 1 | Arch Linux vía `10.10.0.2` | Whitelist activa |
| `1025` | TCP | Java Modpack 2 | Arch Linux vía `10.10.0.2` | Whitelist activa |
| `1026` | TCP | Java Extra | Arch Linux vía `10.10.0.2` | Whitelist activa |
| `1027` | TCP | Java Vanilla 24/7 | Oracle Linux | Público |
| `1028` | UDP | Bedrock Modpack | Arch Linux vía `10.10.0.2` | Público |
| `1029` | UDP | Bedrock Vanilla 24/7 | Oracle Linux | Público |

### Flujo de tráfico

Los servicios alojados en Arch Linux siguen el siguiente flujo:

```text
Jugador
   │
   │ Internet
   ▼
Oracle Cloud
   │
   │ Puerto público
   ▼
Oracle Linux
   │
   │ Firewall + Forwarding
   ▼
WireGuard
   │
   │ 10.10.0.0/24
   ▼
Arch Linux
   │
   │ Puerto interno
   ▼
Servidor Minecraft
```

Esto permite que Oracle Linux funcione como **frontera de seguridad y punto de entrada**, mientras que Arch Linux permanece como infraestructura de cómputo interna.

---

## 3. Enrutamiento mediante WireGuard

El tráfico destinado a los servicios alojados en Arch Linux ingresa inicialmente por Oracle Cloud.

Oracle Linux procesa estas conexiones mediante `firewalld` y las redirige hacia la dirección privada del nodo Arch Linux.

El flujo lógico es:

```text
Internet
   │
   ▼
Oracle Cloud
   │
   ▼
Oracle Linux
   │
   ├── :1024 ──► 10.10.0.2 ──► Java Modpack 1
   ├── :1025 ──► 10.10.0.2 ──► Java Modpack 2
   ├── :1026 ──► 10.10.0.2 ──► Java Extra
   │
   ├── :1027 ──► Java Vanilla
   │
   ├── :1028/UDP ──► 10.10.0.2 ──► Bedrock Modpack
   │
   └── :1029/UDP ──► Bedrock Vanilla
```

Para los servicios alojados en Arch Linux, `firewalld` utiliza reglas de `forward-port` para redirigir el tráfico recibido hacia la dirección privada del túnel:

```text
Puerto público
      │
      ▼
Oracle Linux
      │
      │ firewalld
      │ add-forward-port
      ▼
10.10.0.2
      │
      ▼
Servidor Minecraft
```

La dirección `10.10.0.2` únicamente tiene significado dentro de la red privada de WireGuard.

---

## 4. Aislamiento del nodo de cómputo

Una de las propiedades principales de esta arquitectura es que **Arch Linux no necesita estar expuesto directamente a Internet**.

La separación de responsabilidades queda definida de la siguiente manera:

| Función | Oracle Linux | Arch Linux |
|---|:---:|:---:|
| IP pública | ✓ | — |
| Punto de entrada | ✓ | — |
| Firewall perimetral | ✓ | ✓ |
| Endpoint WireGuard | ✓ | ✓ |
| Reenvío de tráfico | ✓ | — |
| Ejecución de servidores Minecraft | ✓ | ✓ |
| Cómputo principal de Modpacks | — | ✓ |
| Exposición directa a Internet | Limitada | No |

Este modelo permite mantener una frontera clara entre:

**Internet → Gateway → Red privada → Nodo de cómputo**

En consecuencia, una modificación de los servicios de Minecraft no requiere necesariamente modificar la exposición pública del nodo de cómputo.

---

## 5. Consideraciones de seguridad

La arquitectura no depende de una única medida de protección. La seguridad se construye mediante varias capas:

### Segmentación de red

WireGuard proporciona una red privada independiente para la comunicación entre los nodos.

```text
Internet
   │
   ▼
[ Oracle Linux ]
   │
   │ WireGuard
   ▼
[ Red privada ]
   │
   ▼
[ Arch Linux ]
```

### Firewall

El firewall de Oracle Linux debe permitir únicamente los puertos y protocolos necesarios.

Los servicios que no forman parte de la infraestructura pública no deben exponerse.

### Reducción de superficie de ataque

Arch Linux no necesita una dirección pública para prestar los servicios de Minecraft alojados en él.

Esto reduce los servicios directamente expuestos y concentra el control del tráfico en Oracle Linux.

### Whitelist de Minecraft

Los servidores Java que utilizan whitelist deben mantenerla habilitada cuando el modelo de acceso del servidor lo permita.

La whitelist constituye un control a nivel de aplicación y complementa, pero no sustituye, las reglas de firewall y segmentación de red.

### Separación de funciones

Oracle Linux se utiliza como **gateway**, mientras que Arch Linux se utiliza principalmente como **nodo de cómputo**.

Esta separación facilita:

- El mantenimiento de la infraestructura.
- El análisis de tráfico.
- La aplicación de reglas de firewall.
- La detección de conexiones inesperadas.
- La migración o sustitución del nodo de cómputo.

---

## 6. Modelo de confianza

La arquitectura debe asumir que **Internet no es una red confiable**.

Por ello:

1. El tráfico externo llega primero al nodo perimetral.
2. El firewall determina qué tráfico puede continuar.
3. WireGuard proporciona el canal privado entre los nodos.
4. Arch Linux solo recibe el tráfico que ha sido encaminado hacia él.
5. Los propios servidores de Minecraft aplican controles adicionales, como whitelists y autenticación.

La seguridad, por tanto, no depende únicamente de ocultar la dirección del nodo de cómputo.

El objetivo es mantener una arquitectura donde cada capa tenga una responsabilidad concreta:

```text
┌─────────────────────────────────────┐
│ Internet                            │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Oracle Cloud / Oracle Linux         │
│                                     │
│  Firewall                           │
│  Routing                            │
│  Forwarding                         │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ WireGuard                           │
│                                     │
│  Red privada 10.10.0.0/24           │
└─────────────────┬───────────────────┘
                  │
                  ▼
┌─────────────────────────────────────┐
│ Arch Linux                          │
│                                     │
│  Podman rootless                    │
│  Servidores Minecraft               │
│  Datos y mundos                     │
└─────────────────────────────────────┘
```

---

## 7. Principio de diseño

El principio fundamental de esta infraestructura es:

> **El nodo de cómputo no debe asumir la responsabilidad de ser también el perímetro de seguridad.**

Oracle Linux concentra la exposición pública y el enrutamiento, mientras que Arch Linux se mantiene como un nodo de cómputo privado.

Esto permite que la infraestructura pueda crecer sin convertir cada servidor de Minecraft en un nuevo punto de exposición pública.

La arquitectura resultante separa tres responsabilidades:

**Exposición → Enrutamiento → Cómputo**

Esta separación constituye la base para añadir posteriormente nuevas capas de seguridad, monitorización, proxies o nodos de cómputo sin rediseñar completamente la red.
