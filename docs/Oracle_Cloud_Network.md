# Oracle Cloud — Gateway y Punto de Entrada Público

## Propósito

La instancia de **Oracle Cloud** constituye el perímetro público de la infraestructura.

Su función es recibir las conexiones procedentes de Internet, aplicar los controles de acceso correspondientes y transportar el tráfico autorizado hacia los servicios internos mediante el túnel **WireGuard**.

En esta arquitectura, Oracle Linux funciona como:

- Punto de entrada público.
- Gateway de red.
- Firewall del sistema operativo.
- Nodo de terminación del túnel WireGuard.
- Punto de reenvío hacia el nodo de cómputo.
- Capa de aislamiento entre Internet y Arch Linux.

**Dirección WireGuard:** `10.10.0.1`

El diseño general es:

```text
                         INTERNET
                            │
                            ▼
                 ┌────────────────────┐
                 │    Oracle Cloud     │
                 │                     │
                 │  VCN / Security     │
                 │       Lists         │
                 └─────────┬──────────┘
                           │
                           ▼
                 ┌────────────────────┐
                 │    Oracle Linux    │
                 │                    │
                 │    firewalld       │
                 │    Routing         │
                 │    NAT             │
                 └─────────┬──────────┘
                           │
                      WireGuard
                       10.10.0.1
                           │
                           ▼
                 ┌────────────────────┐
                 │    Arch Linux      │
                 │    10.10.0.2       │
                 │                    │
                 │  Podman / Minecraft│
                 └────────────────────┘
```

---

## 1. Oracle Cloud VCN — Security Lists

Antes de que el tráfico llegue al sistema operativo, Oracle Cloud aplica las reglas de seguridad asociadas a la **VCN**.

Estas reglas constituyen el primer filtro de tráfico de entrada.

La configuración se encuentra en:

```text
VCN
└── Security Lists
    └── Default Security List
        └── Ingress Rules
```

Los puertos públicos utilizados por los servidores deben estar permitidos explícitamente.

### Servidores Java

Los servidores Java utilizan TCP:

| Parámetro | Valor |
|---|---|
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | `TCP` |
| Destination Port Range | `1024-1026` |

Esta regla permite que los jugadores establezcan conexiones TCP hacia los tres servidores Java publicados.

### Servidor Bedrock

El servidor Bedrock utiliza UDP:

| Parámetro | Valor |
|---|---|
| Source CIDR | `0.0.0.0/0` |
| IP Protocol | `UDP` |
| Destination Port Range | `1028` |

> **Principio de mínimo privilegio:** únicamente deben abrirse los puertos que correspondan a servicios realmente publicados. Los puertos de administración no deben exponerse públicamente si pueden restringirse mediante WireGuard u otra red de administración.

---

## 2. Capas de filtrado

La exposición pública pasa por varias capas independientes:

```text
Internet
   │
   ▼
┌──────────────────────────┐
│ Oracle Cloud VCN         │
│                          │
│ Security Lists           │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Oracle Linux             │
│                          │
│ firewalld                │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ WireGuard                │
│                          │
│ 10.10.0.0/24             │
└────────────┬─────────────┘
             │
             ▼
┌──────────────────────────┐
│ Arch Linux               │
│                          │
│ Podman / Minecraft       │
└──────────────────────────┘
```

Cada capa cumple una función distinta:

**VCN → Firewall → Routing/NAT → VPN → Servicio**

El objetivo es evitar que la disponibilidad de un servicio implique automáticamente acceso directo al nodo de cómputo.

---

## 3. Reenvío IPv4

Oracle Linux debe poder reenviar paquetes entre sus interfaces de red.

El reenvío IPv4 puede habilitarse temporalmente mediante:

```bash
sudo sysctl -w net.ipv4.ip_forward=1
```

Para comprobar el estado:

```bash
sysctl net.ipv4.ip_forward
```

La salida esperada es:

```text
net.ipv4.ip_forward = 1
```

Para que esta configuración persista después de un reinicio, debe definirse mediante la configuración persistente de `sysctl`.

Por ejemplo:

```text
/etc/sysctl.d/99-forwarding.conf
```

con:

```conf
net.ipv4.ip_forward = 1
```

Y posteriormente:

```bash
sudo sysctl --system
```

---

## 4. NAT y enmascaramiento

El tráfico que entra desde Internet tiene como origen la dirección del jugador.

Cuando Oracle Linux lo reenvía hacia Arch Linux mediante WireGuard, el nodo interno necesita poder devolver correctamente las respuestas.

La infraestructura utiliza **NAT** para que el tráfico que atraviesa el túnel pueda mantener un camino de retorno coherente.

El enmascaramiento puede habilitarse en `firewalld` mediante:

```bash
sudo firewall-cmd --permanent --zone=public --add-masquerade
```

Después:

```bash
sudo firewall-cmd --reload
```

> La regla de NAT debe entenderse dentro del diseño completo de enrutamiento. No sustituye las reglas de firewall ni las rutas necesarias entre las interfaces.

---

## 5. Corrección del enrutamiento asimétrico

Uno de los aspectos críticos de este diseño es el **camino de retorno del tráfico**.

Sin NAT, un paquete puede seguir este recorrido:

```text
Jugador
   │
   ▼
Oracle Linux
   │
   │ WireGuard
   ▼
Arch Linux
```

pero la respuesta podría intentar salir directamente desde Arch Linux hacia Internet:

```text
Arch Linux
   │
   ▼
Internet
```

Esto produciría un camino de retorno diferente al utilizado por la conexión original.

Para evitar esta situación, Oracle Linux puede realizar `MASQUERADE` sobre el tráfico que abandona el sistema mediante `wg0`.

La regla utilizada es:

```bash
sudo firewall-cmd --permanent \
  --direct \
  --add-rule ipv4 nat POSTROUTING 0 \
  -o wg0 \
  -j MASQUERADE
```

Después de modificar la configuración:

```bash
sudo firewall-cmd --reload
```

El flujo resultante es:

```text
Jugador
   │
   │ IP pública
   ▼
Oracle Linux
   │
   │ NAT / MASQUERADE
   │
   │ WireGuard
   ▼
Arch Linux
   │
   │ Respuesta
   ▼
Oracle Linux
   │
   │ NAT inverso
   ▼
Jugador
```

De esta manera, Arch Linux puede tratar a Oracle Linux como el origen visible de la conexión dentro del túnel, manteniendo el retorno del tráfico dentro de la arquitectura definida.

---

## 6. Forwarding de puertos

Los puertos publicados en Oracle Cloud se corresponden con los servicios internos mediante reglas de forwarding.

El modelo general es:

```text
Puerto público
      │
      ▼
Oracle Linux
      │
      │ firewalld
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

Por ejemplo:

```text
TCP :1024
    │
    ▼
10.10.0.2
    │
    ▼
Minecraft Java Modpack 1
```

```text
TCP :1025
    │
    ▼
10.10.0.2
    │
    ▼
Minecraft Java Modpack 2
```

```text
TCP :1026
    │
    ▼
10.10.0.2
    │
    ▼
Minecraft Java Extra
```

```text
UDP :1028
    │
    ▼
10.10.0.2
    │
    ▼
Minecraft Bedrock Modpack
```

Los detalles de cada regla de `forward-port` deben mantenerse sincronizados con la matriz de puertos definida en la documentación de red.

---

## 7. Verificación del estado de la infraestructura

Después de realizar cambios en el firewall o en el enrutamiento, es conveniente verificar cada capa individualmente.

### Estado de firewalld

```bash
sudo firewall-cmd --state
```

### Reglas permanentes

```bash
sudo firewall-cmd --list-all
```

### Reglas directas

```bash
sudo firewall-cmd --direct --get-all-rules
```

### Estado del reenvío IPv4

```bash
sysctl net.ipv4.ip_forward
```

### Estado de WireGuard

```bash
sudo wg show
```

### Puertos en escucha

```bash
sudo ss -tulpen
```

La comprobación debe realizarse desde el exterior y desde el propio nodo, ya que un servicio puede estar correctamente escuchando localmente pero no ser accesible debido a una regla de firewall, Security List o forwarding incorrecto.

---

## 8. Principio de seguridad

Oracle Cloud no debe considerarse simplemente como un servidor adicional.

Dentro de esta arquitectura representa la **frontera entre la red pública y la infraestructura privada**.

Su responsabilidad es mantener una separación clara:

```text
                    ZONA PÚBLICA
                         │
                         ▼
              ┌────────────────────┐
              │   Oracle Cloud     │
              │                    │
              │ Security Lists     │
              │ firewalld          │
              │ NAT                │
              │ Routing            │
              └─────────┬──────────┘
                        │
                   WireGuard
                        │
                        ▼
                  ZONA PRIVADA
              ┌────────────────────┐
              │    Arch Linux      │
              │                    │
              │ Podman Rootless    │
              │ Minecraft          │
              │ Datos persistentes │
              └────────────────────┘
```

El diseño sigue un principio fundamental:

> **La exposición pública debe terminar en el gateway; el nodo de cómputo debe permanecer dentro de la red privada.**

Esto permite concentrar las decisiones de entrada, filtrado, NAT y enrutamiento en Oracle Linux, mientras Arch Linux se mantiene enfocado en su función principal: ejecutar los servicios de Minecraft.
