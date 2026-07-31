---
read_when:
    - Necesita la descripción general de la arquitectura de red y la seguridad
    - Está depurando el acceso local frente al acceso mediante tailnet o el emparejamiento
    - Quieres la lista canónica de documentos sobre redes
summary: 'Centro de red: superficies del Gateway, emparejamiento, descubrimiento y seguridad'
title: Red
x-i18n:
    generated_at: "2026-07-26T05:44:50Z"
    model: gpt-5.6
    postprocess_version: locale-links-v1
    prompt_version: 32
    provider: openai
    source_hash: 9751bb0fe71009455b243b109ef7ef4eda08d58f940f7dcef305800a5ed89586
    source_path: network.md
    workflow: 16
---

Este centro enlaza la documentación principal sobre cómo OpenClaw conecta, empareja y protege
dispositivos en localhost, la LAN y la tailnet.

## Modelo principal

La mayoría de las operaciones pasan por el Gateway (`openclaw gateway`), un único proceso de larga ejecución que gestiona las conexiones de los canales y el plano de control WebSocket.

- **Primero, loopback**: el WS del Gateway usa de forma predeterminada `ws://127.0.0.1:18789`.
  Los enlaces que no son loopback se niegan a iniciarse sin una ruta válida de autenticación del Gateway:
  autenticación mediante token de secreto compartido o contraseña, o una implementación
  `trusted-proxy` que no sea loopback correctamente configurada.
- Se recomienda **un Gateway por host**. Para aislarlos, se pueden ejecutar varios gateways con perfiles y puertos independientes ([Varios Gateways](/es/gateway/multiple-gateways)).
- El **host de Canvas** se sirve en el mismo puerto que el Gateway (`/__openclaw__/canvas/`, `/__openclaw__/a2ui/`) y está protegido por la autenticación del Gateway cuando se enlaza más allá de loopback.
- El **acceso remoto** suele realizarse mediante un túnel SSH o una VPN de Tailscale ([Acceso remoto](/es/gateway/remote)).

Referencias clave:

- [Arquitectura del Gateway](/es/concepts/architecture)
- [Protocolo del Gateway](/es/gateway/protocol)
- [Guía operativa del Gateway](/es/gateway)
- [Superficies web y modos de enlace](/es/web)

## Emparejamiento e identidad

- [Descripción general del emparejamiento (mensajes directos y nodos)](/es/channels/pairing)
- [Emparejamiento de nodos gestionado por el Gateway](/es/gateway/pairing)
- [CLI de dispositivos (emparejamiento y rotación de tokens)](/es/cli/devices)
- [CLI de emparejamiento (aprobaciones de mensajes directos)](/es/cli/pairing)

Confianza local:

- Las conexiones locales directas mediante loopback (sin encabezados reenviados ni de proxy) se pueden
  aprobar automáticamente para el emparejamiento a fin de facilitar la experiencia de usuario en el mismo host.
- OpenClaw también dispone de una ruta limitada de conexión consigo mismo, local al backend o contenedor, para
  flujos auxiliares de confianza con secreto compartido.
- Los clientes de la tailnet y la LAN, incluidos los enlaces de tailnet en el mismo host, siguen requiriendo
  la aprobación explícita del emparejamiento.

## Detección y transportes

- [Detección y transportes](/es/gateway/discovery)
- [Bonjour/mDNS](/es/gateway/bonjour)
- [Acceso remoto (SSH)](/es/gateway/remote)
- [Tailscale](/es/gateway/tailscale)

## Nodos y transportes

- [Descripción general de los nodos](/es/nodes)
- [Protocolo de puente (nodos heredados, histórico)](/es/gateway/bridge-protocol)
- [Guía operativa de Node: iOS](/es/platforms/ios)
- [Guía operativa de Node: Android](/es/platforms/android)

## Seguridad

- [Descripción general de la seguridad](/es/gateway/security)
- [Referencia de configuración del Gateway](/es/gateway/configuration)
- [Solución de problemas](/es/gateway/troubleshooting)
- [Doctor](/es/gateway/doctor)

## Contenido relacionado

- [Guía operativa del Gateway](/es/gateway)
- [Acceso remoto](/es/gateway/remote)
