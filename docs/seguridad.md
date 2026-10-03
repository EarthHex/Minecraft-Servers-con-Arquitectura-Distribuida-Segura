# Seguridad Base y Firewall

## Propósito

La infraestructura implementa una estrategia de **defensa en profundidad**, combinando controles de red, segmentación mediante WireGuard, filtrado de paquetes, prevención de intrusiones y auditoría del sistema.

El objetivo no es depender de una única barrera de seguridad, sino establecer varias capas independientes:

```text id="q2x1m7"
Internet
   │
   ▼
┌────────────────────────────┐
│ Oracle Cloud               │
│                            │
│ OCI Security Lists         │
└─────────────┬──────────────┘
              │
              ▼
┌────────────────────────────┐
│ Oracle Linux               │
│                            │
│ firewalld                  │
│ NAT / Forwarding           │
│ Fail2ban                   │
│ auditd                     │
└─────────────┬──────────────┘
              │
          WireGuard
              │
              ▼
┌────────────────────────────┐
│ Arch Linux                 │
│                            │
│ nftables                   │
│ Fail2ban                   │
│ auditd                     │
│ Podman Rootless            │
└────────────────────────────┘
```

La arquitectura sigue un principio de **mínimo privilegio y confianza explícita**: cada conexión debe atravesar las capas correspondientes antes de alcanzar un servicio.

> **Nota:** El término *Zero Trust* se utiliza aquí como principio de diseño. La implementación concreta se basa en controles de acceso, segmentación y verificación explícita, no en asumir que una red privada es automáticamente confiable.

---

# 1. Firewalls y control de acceso

La infraestructura utiliza diferentes tecnologías de filtrado según la función de cada nodo.

| Nodo | Tecnología | Función principal |
|---|---|---|
| Oracle Cloud | OCI Security Lists | Filtrado perimetral |
| Oracle Linux | `firewalld` | Firewall, NAT y forwarding |
| Arch Linux | `nftables` | Filtrado local |
| Ambos | Fail2ban | Respuesta automática a determinados patrones de abuso |

Cada capa funciona de manera independiente.

Un error de configuración en una capa no debería implicar automáticamente la exposición completa del nodo.

---

## 1.1 Arch Linux — nftables

Arch Linux utiliza **nftables** como firewall principal.

La política general sigue el principio:

> **Denegar por defecto y permitir únicamente el tráfico necesario.**

Las cadenas `input` y `forward` utilizan una política predeterminada de `drop`.

Conceptualmente:

```text id="yn8o4k"
                  ┌───────────────┐
                  │    Tráfico    │
                  └───────┬───────┘
                          │
                          ▼
                 ┌─────────────────┐
                 │    nftables     │
                 └────────┬────────┘
                          │
                ┌─────────┴─────────┐
                │                   │
             Permitido            Denegado
                │                   │
                ▼                   ▼
             Servicio             DROP
```

El acceso SSH se permite únicamente mediante la regla correspondiente al puerto configurado.

El tráfico proveniente de WireGuard se identifica mediante la interfaz virtual `wg0`.

La regla conceptual:

```text id="g2k5re"
iifname "wg0" accept
```

permite el tráfico recibido a través del túnel.

> **Importante:** permitir todo el tráfico procedente de `wg0` convierte a la interfaz en una zona de confianza amplia. En una implementación más restrictiva, las reglas pueden limitarse por dirección, protocolo y puerto según los servicios realmente necesarios.

Esto permite evolucionar posteriormente desde:

```text
wg0 → permitir
```

hacia:

```text
wg0 → servicio específico → puerto específico → origen autorizado
```

sin modificar la arquitectura general.

---

# 1.2 Oracle Linux — firewalld

Oracle Linux utiliza **firewalld** como capa de filtrado y administración dinámica del firewall.

Su responsabilidad dentro de la arquitectura es mayor que la de un firewall local convencional, ya que también participa en:

- NAT.
- Reenvío de paquetes.
- Exposición de servicios públicos.
- Comunicación mediante WireGuard.
- Encaminamiento hacia Arch Linux.

La interfaz WireGuard:

```text
wg0
```

se integra en la configuración de zonas de `firewalld`.

En el diseño actual, `wg0` se encuentra asociada a una zona de confianza:

```text id="k6t5k0"
trusted
```

Esto simplifica la comunicación entre los nodos de la infraestructura.

Sin embargo, una zona `trusted` implica un nivel de confianza elevado. Por ello, debe utilizarse únicamente cuando el modelo de confianza de la red VPN lo justifique.

Una alternativa más restrictiva consiste en utilizar una zona específica para WireGuard y permitir únicamente los servicios requeridos.

---

# 1.3 Oracle Cloud — Security Lists

Antes de que el tráfico alcance `firewalld`, Oracle Cloud aplica las reglas de seguridad de la VCN.

Los servicios públicos utilizados por la infraestructura incluyen:

| Servicio | Protocolo | Puerto |
|---|---|---:|
| Minecraft Java | TCP | `1024-1027` |
| Minecraft Bedrock | UDP | `1028-1029` |
| WireGuard | UDP | `51820` |

Las reglas deben limitarse exclusivamente a los puertos que realmente estén publicados.

La cadena de filtrado puede representarse como:

```text id="p5u7ki"
Internet
   │
   ▼
OCI Security Lists
   │
   ▼
firewalld
   │
   ▼
WireGuard / Servicio local
   │
   ▼
Minecraft
```

Cada capa debe considerarse un control independiente.

---

# 2. Puertos no convencionales

La infraestructura utiliza algunos puertos externos no convencionales para los servicios de Minecraft y administración.

Esto puede reducir parte del tráfico automatizado generado por herramientas que buscan únicamente servicios en puertos conocidos.

Sin embargo:

> **Cambiar un puerto no constituye una medida de seguridad suficiente.**

Un escáner puede detectar igualmente un servicio aunque utilice un puerto diferente.

Por esta razón, la protección real depende de:

- Firewall.
- Restricción de origen.
- WireGuard.
- Autenticación.
- Whitelist de Minecraft.
- Fail2ban cuando sea aplicable.
- Actualizaciones del sistema.
- Aislamiento mediante contenedores.
- Monitorización y auditoría.

Los puertos no convencionales deben considerarse una medida de **reducción de ruido**, no una barrera de seguridad.

---

# 3. Prevención de intrusiones — Fail2ban

**Fail2ban** proporciona una capa de respuesta automática ante determinados patrones de abuso.

Su funcionamiento general consiste en:

```text id="6hlf8u"
Evento en logs
      │
      ▼
Fail2ban
      │
      ▼
Detección de patrón
      │
      ▼
Exceso de intentos
      │
      ▼
Regla temporal de firewall
      │
      ▼
Bloqueo del origen
```

Fail2ban no reemplaza al firewall.

Su función es reaccionar dinámicamente ante eventos detectados en los registros.

---

## 3.1 Oracle Linux

Debido a la base Enterprise Linux utilizada por Oracle Linux, Fail2ban puede instalarse mediante **EPEL (Extra Packages for Enterprise Linux)** cuando el paquete no esté disponible en los repositorios habilitados por defecto.

Los componentes utilizados por la infraestructura incluyen:

```text id="o4i9vu"
fail2ban
fail2ban-firewalld
fail2ban-systemd
```

La configuración se mantiene modularmente mediante:

```text id="z5k9uo"
/etc/fail2ban/jail.d/sshd.local
```

La política documentada actualmente establece:

| Parámetro | Valor |
|---|---:|
| Intentos permitidos | 5 |
| Ventana de observación | 10 minutos |
| Duración del bloqueo | 24 horas |
| Acción | `firewallcmd-rich-rules` |

El objetivo es limitar ataques automatizados contra SSH.

La política puede representarse como:

```text id="49q7jm"
5 intentos fallidos
       │
       ▼
Dentro de 10 minutos
       │
       ▼
Fail2ban detecta el patrón
       │
       ▼
Firewall bloquea el origen
       │
       ▼
24 horas
```

Los valores deben considerarse parámetros operativos y ajustarse según los logs reales y las necesidades de administración.

---

# 3.2 Arch Linux

Arch Linux utiliza Fail2ban para analizar los registros generados por `systemd-journald`.

El objetivo es detectar intentos repetitivos de autenticación fallida y aplicar bloqueos temporales cuando corresponda.

El flujo es:

```text id="z1v8r6"
sshd
 │
 ▼
systemd-journald
 │
 ▼
Fail2ban
 │
 ▼
nftables / firewall
 │
 ▼
Bloqueo
```

Este mecanismo permite proteger SSH independientemente de que el intento de conexión proceda de una red externa o de un segmento accesible mediante la VPN.

---

# 4. Auditoría del sistema — auditd

La infraestructura utiliza **auditd** para registrar eventos relevantes a nivel del sistema operativo.

Mientras Fail2ban está orientado principalmente a la **respuesta**, `auditd` está orientado a la **trazabilidad**.

La diferencia conceptual es:

```text id="yq6q9v"
Fail2ban
   │
   └── Detectar → Responder

auditd
   │
   └── Registrar → Auditar → Investigar
```

---

## 4.1 Archivos críticos

La auditoría se concentra en archivos relacionados con autenticación y configuración de infraestructura.

Entre ellos:

```text id="tq2bbi"
/etc/passwd
/etc/shadow
/etc/ssh/sshd_config
```

También pueden incluirse archivos relacionados con la configuración del firewall y de red cuando sea necesario.

El objetivo es poder determinar:

- Qué archivo fue modificado.
- Cuándo ocurrió la modificación.
- Qué proceso realizó la operación.
- Qué usuario estaba asociado con la acción.
- Qué cambios requieren investigación.

---

# 5. Monitorización y respuesta

La seguridad de esta infraestructura no termina con la configuración inicial del firewall.

Los controles deben poder verificarse periódicamente.

Algunos puntos relevantes son:

```text id="l3d5h6"
Firewall
   │
   ├── Reglas activas
   ├── Puertos publicados
   └── Interfaces autorizadas

Fail2ban
   │
   ├── Jails activos
   ├── IPs bloqueadas
   └── Eventos recientes

auditd
   │
   ├── Cambios de configuración
   ├── Cambios de autenticación
   └── Eventos del sistema

WireGuard
   │
   ├── Peers
   ├── Handshakes
   └── Transferencia
```

La revisión periódica permite detectar configuraciones que hayan quedado obsoletas y reducir progresivamente la superficie de ataque.

---

# 6. Modelo de defensa en profundidad

La seguridad de la infraestructura se basa en varias capas:

| Capa | Mecanismo | Objetivo |
|---|---|---|
| Perímetro cloud | OCI Security Lists | Filtrar tráfico antes del host |
| Host público | `firewalld` | Controlar acceso y forwarding |
| VPN | WireGuard | Crear canal privado entre nodos |
| Host interno | `nftables` | Filtrar tráfico local |
| Aplicación | Minecraft / whitelist | Controlar acceso al servicio |
| Contenedores | Podman Rootless | Reducir privilegios de procesos |
| Detección | Fail2ban | Responder a patrones de abuso |
| Auditoría | `auditd` | Mantener trazabilidad de eventos |

El modelo completo puede resumirse como:

```text id="e0kq0j"
              INTERNET
                  │
                  ▼
        ┌───────────────────┐
        │ OCI Security List │
        └─────────┬─────────┘
                  │
                  ▼
        ┌───────────────────┐
        │    firewalld      │
        │       + NAT       │
        └─────────┬─────────┘
                  │
                  ▼
             WireGuard
                  │
                  ▼
        ┌───────────────────┐
        │     nftables      │
        └─────────┬─────────┘
                  │
                  ▼
        ┌───────────────────┐
        │   Podman Rootless │
        └─────────┬─────────┘
                  │
                  ▼
        ┌───────────────────┐
        │    Minecraft      │
        │     Whitelist     │
        └───────────────────┘

       ┌───────────────────────┐
       │ Fail2ban + auditd     │
       │ detección y auditoría │
       └───────────────────────┘
```

La arquitectura resultante sigue un principio sencillo:

> **Reducir la superficie expuesta, limitar los privilegios, controlar explícitamente el tráfico y mantener evidencia suficiente para investigar eventos de seguridad.**

Este enfoque permite que la infraestructura evolucione sin depender de una única tecnología de protección y proporciona una base para incorporar posteriormente controles adicionales de endurecimiento, monitorización y respuesta.
