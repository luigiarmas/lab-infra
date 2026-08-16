# lab-infra

Documentación de la topología del laboratorio de virtualización usado durante el
programa de formación en Cloud Security Engineering.

## Objetivo

Registrar de forma reproducible la infraestructura base sobre la que se ejecutan
los laboratorios de administración de sistemas, automatización, hardening y
seguridad en la nube.

## Componentes

| Nodo | Tipo | Sistema operativo | Rol |
|---|---|---|---|
| `rocky-lab` | VM | Rocky Linux 9.8 minimal | Objetivo Linux, nodo gestionado por Ansible |
| `winserver-lab` | VM | Windows Server 2022 Standard | Servicios de directorio, objetivo SMB |
| `parrot-lab` | VM | Parrot OS | Estación de auditoría |
| `ubuntu-node` | Físico | Ubuntu Server LTS | Nodo gestionado, sujeto de hardening CIS |

## Hipervisor

VirtualBox 7.2.8 sobre Windows 11 Pro.

Configuración común a todas las máquinas virtuales:

- Firmware **UEFI** habilitado antes de la instalación
- Disco virtual de reserva dinámica
- Adaptador de red en **modo puente**
- Instantánea `base-post-install` tomada tras la instalación y actualización inicial

## Red

Todos los nodos operan sobre un mismo segmento de capa 2, con direccionamiento
asignado por DHCP. El modo puente permite comunicación directa entre máquinas
virtuales y nodos físicos, requisito para la gestión por SSH y la ejecución de
playbooks de automatización.

## Notas

Los identificadores, direcciones y nombres de host de este repositorio
corresponden a un laboratorio de formación. No representan infraestructura
productiva de ninguna organización.

## Licencia

MIT — ver [LICENSE](LICENSE).

## Estado del proyecto
En desarrollo activo. Semana 2 de 72 del plan de formación en Cloud Security Engineering.

