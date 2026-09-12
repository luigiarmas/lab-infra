# Red del laboratorio

Documentación de la topología de red, direccionamiento, rutas y reglas de
firewall del laboratorio de virtualización.

## Topología

~~
                        Internet
                            |
                   +--------+--------+
                   |  Router / GW    |
                   |  192.168.18.1   |
                   |  (gateway+DNS)  |
                   +--------+--------+
                            |
              ==============+==============  192.168.18.0/24
                            |
        +-------------------+-------------------+
        |                   |                   |
+-------+-------+   +-------+-------+   +-------+-------+
|   ThinkPad    |   |  rocky-lab    |   | winserver-lab |
|   (host)      |   |  .37 (DHCP)   |   |  .38 (DHCP)   |
|   Windows 11  |   |  Rocky 9.8    |   |  WS 2022      |
+---------------+   +---------------+   +---------------+
                            |
                    +-------+-------+
                    |  ubuntu-node  |
                    |  (planificado)|
                    |  Dell fisico  |
                    +---------------+
~~

Todas las VMs usan adaptador **puente** sobre la interfaz física del host, por lo
que obtienen IP del router en el mismo segmento que el resto de equipos.

## Direccionamiento

| Nodo | IP | Método | Interfaz |
|---|---|---|---|
| Router / gateway | 192.168.18.1 | - | - |
| rocky-lab | 192.168.18.37/24 | DHCP | enp0s3 |
| winserver-lab | 192.168.18.38/24 | DHCP | - |
| ubuntu-node | pendiente | - | - |

Segmento: `192.168.18.0/24`. Máscara `/24` = los primeros tres octetos identifican
la red; todos los `192.168.18.x` se alcanzan directamente sin pasar por el gateway.

## Rutas (rocky-lab)

~~
default via 192.168.18.1 dev enp0s3 proto dhcp
192.168.18.0/24 dev enp0s3 proto kernel scope link
~~

- Tráfico a `192.168.18.x` -> directo por `enp0s3` (scope link)
- Todo lo demás -> gateway `192.168.18.1`

Verificado con `traceroute 8.8.8.8`: primer salto `192.168.18.1`.

## DNS

Resolver: `192.168.18.1` (el router). Configurado por DHCP y gestionado por
NetworkManager: `/etc/resolv.conf` se regenera automáticamente, no se edita a mano.

La red tiene IPv6 activo; `getent hosts` prefiere IPv6 cuando está disponible.

## Servicios expuestos (rocky-lab)

Resultado de `ss -tulnp`:

| Servicio | Puerto | Dirección | Alcance |
|---|---|---|---|
| sshd | 22/tcp | 0.0.0.0 | Toda la red |
| chronyd | 323/udp | 127.0.0.1 | Solo local |

`sshd` escucha en todas las interfaces porque se administra remotamente.
`chronyd` en loopback: no expuesto.

## Firewall (rocky-lab)

Backend: `firewalld` sobre `nftables`. Zona activa: `public` (interfaz `enp0s3`).

**Servicios permitidos:**

| Servicio | Puerto | Justificación |
|---|---|---|
| ssh | 22/tcp | Administración remota |
| dhcpv6-client | 546/udp | Negociación IPv6 |

**Política por defecto:** `reject with icmpx admin-prohibited`. Todo lo no
permitido se rechaza con aviso al origen.

**ICMP:** permitido (la VM responde a ping).

### Cambios aplicados

| Fecha | Cambio | Razón |
|---|---|---|
| 2026-09-10 | Eliminado `cockpit` (9090/tcp) | Servicio no instalado; regla sin propósito ampliaba superficie |

### Regla nftables resultante

~~
chain filter_IN_public_allow {
    tcp dport 22 accept
    ip6 daddr fe80::/64 udp dport 546 accept
}
~~

## Tráfico observado en el segmento

Captura con `tcpdump -i enp0s3 -n 'not port 22'`:

- ICMP echo periódico desde el router hacia los equipos
- Broadcast UDP en puerto 15600 (descubrimiento de dispositivos multimedia)
- Multicast SSDP a `239.255.255.250`
- Broadcast Ethernet no estándar (`ethertype 0x8300`) en ráfagas de 8 paquetes
  por segundo, origen `04:cc:bc:xx:xx:xx`

El segmento es compartido con dispositivos domésticos. En entorno productivo
esto se resolvería con segmentación en VLANs.

## Pendientes

- [ ] Asignar IP y documentar `ubuntu-node` (semana 5)
- [ ] Tailscale en todos los nodos para direccionamiento independiente del medio
      físico (el puente se ata a una NIC concreta: cambiar de cable a Wi-Fi rompe
      la conectividad)
- [ ] Evaluar bloqueo de ICMP en hardening (semana 11)
- [ ] Reservar IPs por MAC en el router o pasar a estáticas para estabilidad del
      inventario de Ansible (semana 7)
