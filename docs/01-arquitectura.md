# Arquitectura de Red y Túnel Seguro (WireGuard)

La infraestructura utiliza una topología Hub-and-Spoke donde **Oracle Linux** actúa como el nodo perimetral de entrada y **Arch Linux** permanece en una red privada sin exposición pública directa.

## Red Virtual Privada (WireGuard)

* **Subred Interna:** `10.10.0.0/24`
* **Oracle Linux (Hub):** IP VPN `10.10.0.1` | Puerto Listening: `<WG_PORT>/UDP`
* **Arch Linux (Spoke):** IP VPN `10.10.0.2` | PersistentKeepalive: 25s

## Gestión de Nombres (No-IP)
* **Proveedor DDNS:** No-IP mediante contenedor en Podman.
* **Ubicación del agente:** Oracle Linux (Bastión).
* **Función:** Actualización automática del registro A con la IP pública asignada por Oracle Cloud.

## Runtime de Contenedores
* **Motor:** Podman (Rootless).
* **Orquestación:** `podman-compose`.
