# Arquitectura de Red y Túnel Seguro (WireGuard)

La infraestructura utiliza una topología Hub-and-Spoke. **Oracle Linux** actúa como el bastión público (Hub) y enruta el tráfico hacia **Arch Linux** (Spoke), el cual opera como una caja negra sin exposición pública directa.

## Topología de Red Privada
* **Subred VPN:** `10.10.0.0/24`
* **Oracle Linux (Hub):** IP `10.10.0.1` | Endpoint público.
* **Arch Linux (Spoke):** IP `10.10.0.2` | PersistentKeepalive: 25s.

## Matriz de Puertos y Servicios
Se utilizan puertos no convencionales para mitigar escaneos automatizados en los servidores Modpack.

| Puerto Público (OCI) | Protocolo | Servidor Destino | Host Físico | Acceso / Seguridad |
| :--- | :--- | :--- | :--- | :--- |
| **1024** | TCP | Java Modpack 1 | Arch (vía `10.10.0.2`) | Whitelist activa |
| **1025** | TCP | Java Modpack 2 | Arch (vía `10.10.0.2`) | Whitelist activa |
| **1026** | TCP | Java Extra | Arch (vía `10.10.0.2`) | Whitelist activa |
| **1027** | TCP | Java Vanilla 24/7 | Oracle Linux (Local) | Público |
| **1028** | UDP | Bedrock Modpack | Arch (vía `10.10.0.2`) | Público |
| **1029** | UDP | Bedrock Vanilla 24/7 | Oracle Linux (Local) | Público |

## Enrutamiento
El tráfico que ingresa a Oracle Cloud por los puertos `1024-1028` es redirigido nativamente mediante `firewalld (add-forward-port)` hacia la IP interna de WireGuard (`10.10.0.2`).
