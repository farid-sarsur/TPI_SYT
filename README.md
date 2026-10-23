# Proyecto Integrador — Raspberry Pi 3

## Introducción

El presente proyecto integrador tiene como finalidad implementar y configurar una **Raspberry Pi 3 como servidor de servicios de red**, utilizando un sistema operativo basado en Linux y distintas herramientas de administración.

El trabajo se desarrolla de manera progresiva, incorporando servicios que permiten administrar remotamente el dispositivo, ejecutar aplicaciones gráficas desde computadoras cliente y gestionar la asignación de direcciones IP dentro de una red de área local (LAN).

La documentación se encuentra organizada en informes técnicos independientes, correspondientes a cada etapa del proyecto.

## Objetivos del proyecto

- Instalar y configurar un sistema operativo Linux sin entorno de escritorio.
- Implementar el protocolo SSH para administrar remotamente la Raspberry Pi.
- Configurar el servidor gráfico Xorg y habilitar la ejecución remota de aplicaciones gráficas mediante X11 Forwarding.
- Instalar y configurar un servidor DHCP para administrar el direccionamiento IP de una red local.
- Integrar los diferentes servicios en una infraestructura de red funcional.

## Hardware y software utilizado

**Hardware:**
- Raspberry Pi 3.
- Tarjeta microSD.
- Switch Ethernet.
- Cables de red RJ45.
- Computadoras cliente.

**Software y herramientas:**
- Raspberry Pi OS.
- OpenSSH.
- Xorg.
- Xming.
- Servidor DHCP.
- Cisco Packet Tracer.

## Documentación técnica

| Apartado | Contenido | Acceso |
|---|---|---|
| Instalación del sistema operativo y SSH | Instalación de Linux y configuración del acceso remoto seguro | [Ver informe](TP1_OS_SSH/OS_SSH.md) |
| Servidor gráfico Xorg | Instalación de Xorg y configuración de X11 Forwarding mediante SSH | [Ver informe](Xorg_X11_Forwarding/Xorg_X11_Forwarding.md) |
| Servidor DHCP | Configuración del servidor DHCP y diseño de la red LAN | [Ver informe](Servidor-DHCP/) |

## Arquitectura de red

La infraestructura utiliza una Raspberry Pi 3 conectada mediante Ethernet a un switch, que permite interconectar las computadoras del grupo dentro de una misma red local.

La red asignada al Grupo 3 es:

`192.168.3.0/24`

La Raspberry Pi proporciona el servicio DHCP para la asignación automática de direcciones IP a los dispositivos clientes.

El diseño de la infraestructura se documenta mediante un diagrama de red realizado en Cisco Packet Tracer.

## Metodología de trabajo

El proyecto se divide en cinco etapas, cada una acompañada de un informe técnico independiente.

Cada informe contiene una introducción, objetivos, fundamentos teóricos, procedimientos de configuración, explicación de los comandos utilizados y una conclusión sobre los resultados obtenidos.

Al finalizar el proyecto, las cinco etapas se integrarán en un informe técnico final.

## Integrantes

**Grupo 3**

- Kunda, Leandro
- Ledesma, Gabriel
- Medina Lucini, Facundo
- Reynoso, Mateo
- Sarsus, Al Farid

## Estado del proyecto

En desarrollo — tres etapas documentadas de las cinco previstas.
