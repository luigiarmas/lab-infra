# Red del laboratorio

Documentación de la topología de red, direccionamiento, rutas y reglas de
firewall del laboratorio de virtualización.

## Topología

El laboratorio opera sobre dos planos de red simultáneos:

- **Plano físico**: segmento doméstico, direccionamiento por DHCP. La dirección
  cambia al mover los equipos entre redes.
- **Plano de superposición**: red Tailscale (WireGuard) con direcciones fijas en
  el rango 100.64.0.0/10. Independiente de la red física subyacente.

La administración se realiza sobre el plano de superposición. El plano físico se
mantiene para pruebas que requieren un segmento de capa 2 compartido.

    Red de superposicion (100.64.0.0/10) - direcciones fijas

    +---------------+   +---------------+   +----------------+
    |  ThinkPad     |   |  rocky-lab    |   |  agencia-srv   |
    |  Windows 11   |   |  Rocky 9.8    |   |  Ubuntu Server |
    |  hipervisor   |   |  VM           |   |  fisico        |
    +-------+-------+   +-------+-------+   +--------+-------+
            |                   |                    |
            +-------------------+--------------------+
                                |
              Red fisica 192.168.x.0/24 (DHCP, variable)
                                |
                       +--------+--------+
                       |  Router / GW    |
                       |  gateway + DNS  |
                       +-----------------+

Las máquinas virtuales usan adaptador puente, por lo que obtienen dirección del
router en el mismo segmento que los equipos físicos.

## Nodos

| Nodo | Rol | Sistema | Acceso |
|---|---|---|---|
| ThinkPad | Estación de trabajo, hipervisor | Windows 11 Pro | — |
| rocky-lab | Objetivo Linux, nodo gestionado | Rocky Linux 9.8 | alias SSH |
| agencia-srv | Servidor de aplicaciones, nodo gestionado | Ubuntu Server | alias SSH |
| winserver-lab | Servicios de directorio, objetivo SMB | Windows Server 2022 | consola |
| parrot-lab | Estación de auditoría | Parrot OS | consola |

## Direccionamiento

Cada nodo mantiene dos direcciones: una asignada por DHCP en el segmento físico
y otra fija en la red de superposición.

| Nodo | Físico | Superposición | Interfaz física |
|---|---|---|---|
| rocky-lab | DHCP, variable | fija | enp0s3 |
| agencia-srv | DHCP, variable | fija | eno2 (cable), wlo1 (Wi-Fi) |
| ThinkPad | DHCP, variable | fija | Wi-Fi o Ethernet |

El nodo agencia-srv mantiene ambas interfaces físicas activas. El enlace por
cable tiene métrica 100 y el inalámbrico 600, por lo que el cable es la ruta
preferida.

### Motivo de la red de superposición

El adaptador puente de VirtualBox se asocia a una interfaz física concreta. Al
cambiar el equipo anfitrión entre cable y Wi-Fi, o entre redes distintas, la
máquina virtual queda inalcanzable hasta reconfigurar el adaptador y actualizar
la dirección en el cliente SSH.

Con direcciones de superposición fijas, la configuración del cliente se escribe
una vez. Es requisito para que el inventario de automatización se mantenga
estable a lo largo del tiempo.

## Rutas

Ejemplo de un nodo virtual:

    default via <gateway> dev <interfaz> proto dhcp
    <red-local>/24 dev <interfaz> proto kernel scope link
    <red-superposicion> dev tailscale0

- Tráfico al segmento local: directo por la interfaz física (scope link)
- Tráfico al rango de superposición: por la interfaz tailscale0
- Todo lo demás: gateway por defecto

## DNS

Resolver: el router del segmento, asignado por DHCP y gestionado por
NetworkManager. El archivo /etc/resolv.conf se regenera automáticamente; la
configuración se modifica mediante nmcli, no editando el archivo.

La red tiene IPv6 activo. Las utilidades de resolución prefieren IPv6 cuando
está disponible.

## Servicios expuestos

### rocky-lab

| Servicio | Puerto | Dirección | Alcance |
|---|---|---|---|
| sshd | 2222/tcp | 0.0.0.0 | Toda la red |
| chronyd | 323/udp | 127.0.0.1 | Solo local |

### agencia-srv

| Servicio | Puerto | Dirección | Alcance |
|---|---|---|---|
| sshd | 22/tcp | 0.0.0.0 | Toda la red |
| nginx en contenedor | 80/tcp | 0.0.0.0 | Toda la red |

El contenedor edge-nginx, imagen nginx:1.29-alpine, publica el puerto 80 en
todas las interfaces. No hay TLS configurado.

Docker crea la interfaz docker0 en 172.17.0.0/16 y bridges adicionales por cada
red de Compose definida.

## Firewall

### rocky-lab

Backend: firewalld sobre nftables. Zona activa: public en la interfaz física.

| Elemento | Valor |
|---|---|
| Servicios permitidos | dhcpv6-client |
| Puertos permitidos | 2222/tcp |
| Política por defecto | reject with icmpx admin-prohibited |
| ICMP | Permitido |

La interfaz tailscale0 no está asignada explícitamente a una zona, por lo que
firewalld le aplica la zona por defecto. El tráfico de administración pasa sin
configuración adicional.

### Cambios aplicados

| Semana | Nodo | Cambio | Motivo |
|---|---|---|---|
| 4 | rocky-lab | Eliminado servicio cockpit (9090/tcp) | Servicio no instalado; regla sin propósito |
| 5 | rocky-lab | Puerto SSH movido a 2222 | Reducción de ruido de escaneo automatizado |
| 5 | rocky-lab | Eliminado servicio ssh (22/tcp) | Sin servicio detrás tras el cambio de puerto |
| 5 | rocky-lab | Autorizado el puerto nuevo en SELinux | Requisito para que sshd pueda escuchar en él |

## Acceso SSH

Ambos nodos gestionados aceptan únicamente autenticación por clave pública. La
autenticación por contraseña está deshabilitada.

| Nodo | Puerto | PasswordAuthentication | PermitRootLogin |
|---|---|---|---|
| rocky-lab | 2222 | no | no |
| agencia-srv | 22 | no | por defecto |

Procedimiento completo de endurecimiento documentado en el repositorio
ssh-hardening.

### Nota sobre cloud-init en Ubuntu

En agencia-srv, la directiva PasswordAuthentication no proviene de sshd_config
sino de un fragmento en /etc/ssh/sshd_config.d/ generado por cloud-init durante
la instalación.

El archivo principal muestra la directiva comentada, lo que puede inducir a
error al auditar. La configuración efectiva debe consultarse siempre con:

    sudo sshd -T | grep -i passwordauthentication

### Configuración del cliente

Archivo ~/.ssh/config en la estación de trabajo, con un bloque por nodo. Los
alias apuntan a las direcciones de superposición, no a las físicas, de modo que
no requieren mantenimiento al cambiar de red.

## Tráfico observado en el segmento

Captura con tcpdump excluyendo el puerto de administración:

- ICMP echo periódico desde el router hacia los equipos del segmento
- Broadcast UDP en puerto 15600, descubrimiento de dispositivos multimedia
- Multicast SSDP hacia 239.255.255.250
- Broadcast Ethernet no estándar (ethertype 0x8300) en ráfagas de ocho paquetes
  por segundo desde un único origen

El segmento es compartido con dispositivos domésticos. En un entorno productivo
esto se resolvería mediante segmentación en VLAN.

## Consideraciones sobre la red de superposición

La coordinación de la red depende de un servicio externo. El tráfico entre nodos
va cifrado extremo a extremo y las claves privadas no abandonan cada
dispositivo, pero el control de pertenencia a la red lo administra el proveedor.

Es una arquitectura aceptable para un laboratorio. Para infraestructura con
requisitos de cumplimiento, la evaluación debe considerar la dependencia de ese
plano de control externo.

La red de superposición no sustituye al endurecimiento de cada nodo: cualquier
equipo con acceso a la red alcanza todos los servicios expuestos de los demás.

## Pendientes

- [ ] Configurar TLS en el contenedor nginx; actualmente solo HTTP
- [ ] Revisar actualizaciones de la imagen base del contenedor
- [ ] Evaluar bloqueo de ICMP en la fase de endurecimiento
- [ ] Reservar direcciones por MAC o migrar a estáticas en el segmento físico
- [ ] Aplicar PermitRootLogin no en agencia-srv
- [ ] Incorporar los nodos restantes a la red de superposición

## Alcance

Este documento no incluye direcciones concretas, nombres de dominio,
identificadores de red ni material criptográfico de ningún entorno real.
