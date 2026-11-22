# Arquitectura del Network Operations Lab

Fecha: 20/11/2026  
Actualizado: 21/11/2026  
Estado: diseño implementado en las VMs. pfSense configurado y probado; falta decidir el NAT, probar de punta a punta y poner el monitoreo y los incidentes.

> **Aviso de alcance.** Es un laboratorio en VirtualBox, no una red de producción. Se armó con guías y ayuda de una IA. Nada de lo que está aquí es experiencia en producción.

---

## 1. Idea

Una red pequeña protegida por **pfSense**, con un router **MikroTik CHR** detrás, monitoreada por **LibreNMS** y con los incidentes registrados en **GLPI**. El hilo conductor es el ciclo de un NOC (centro de operaciones de red) y de soporte IT: detectar, diagnosticar, resolver y documentar.

## 2. Topología

Tráfico del laboratorio:

```
                        Internet
                           │
                 [NAT de VirtualBox]  10.0.2.0/24
                           │
                     em0  10.0.2.15
                   ┌───────┴───────┐
                   │    pfsense    │
                   └───────┬───────┘
                     em1  192.168.10.1
                           │
 ═════════ lab-core  192.168.10.0/24 ═════════════════
        │                 │                  │
   librenms .13        glpi .14        chr (ether1) .12
                                           │
                                     ┌─────┴─────┐
                                     │    chr    │
                                     └─────┬─────┘
                                  ether2  10.10.10.1
                                           │
 ═════════ lab-clientes  10.10.10.0/24 ═══════════════
                                           │
                                       cliente1
                              (DHCP del CHR: .100 a .200)
                     VLAN 10 (10.10.20.0/24): llega en la etapa de integración
```

Red de gestión, aparte (solo-anfitrión):

```
   PC anfitrión  192.168.56.1
         │
 ═════ gestion  192.168.56.0/24 ═════════════════════════
    │ .11        │ .12         │ .13        │ .14       │ DHCP
 pfsense         chr        librenms      glpi       cliente1
 (em2)         (ether3)     (enp0s8)     (enp0s8)    (enp0s8)
```

## 3. Redes

| Nombre en VirtualBox | Tipo | Rango | Para qué | Quién reparte IP |
|---|---|---|---|---|
| (NAT) | NAT | 10.0.2.0/24 | Salida de pfSense a internet | VirtualBox |
| `lab-core` | Red interna | 192.168.10.0/24 | Núcleo: pfSense, CHR, LibreNMS, GLPI | pfSense (con reservas) |
| `lab-clientes` | Red interna | 10.10.10.0/24 | Clientes detrás del CHR | CHR |
| `gestion` (etiqueta del lab) | Solo-anfitrión (VirtualBox Host-Only Ethernet Adapter) | 192.168.56.0/24 | Entrar por SSH, Winbox y web desde mi PC | IP fija en cada servidor; VirtualBox para el cliente |

El detalle de IPs está en `direccionamiento.md`.

## 4. Equipos

| VM en VirtualBox | Rol | RAM | Adaptadores |
|---|---|---|---|
| `pfsense` | Borde: firewall, NAT, DNS, DHCP y NTP de `lab-core` | 1 GB | NAT, `lab-core`, solo-anfitrión |
| `chr` | Router interno (MikroTik CHR) y DHCP de `lab-clientes` | 1 GB | `lab-core`, `lab-clientes`, solo-anfitrión |
| `librenms` | Monitoreo | 2 GB | `lab-core`, solo-anfitrión |
| `glpi` | Tickets e inventario | 1 GB | `lab-core`, solo-anfitrión |
| `cliente1` | Cliente de prueba | 1 GB | `lab-clientes`, solo-anfitrión |
| `cliente-origen` | VM de la que salió `cliente1` (clon enlazado). **No se enciende ni se borra** | n/a | n/a |

RouterOS del chr: 7.25.1 (verificado el 20/11/2026).

## 5. Decisiones y por qué

**Red de gestión aparte (fuera de banda).** Las VMs de Ubuntu no tienen copiar y pegar en la consola de VirtualBox, así que se administran por SSH desde mi PC. Esa entrada va por una red separada del tráfico que se monitorea. En un NOC real pasa igual: el canal de gestión existe para poder entrar justo cuando falla el servicio principal.

**LibreNMS monitorea por `lab-core`, no por gestión.** LibreNMS marca un equipo como caído cuando no le responde la IP registrada. Si usara la IP de gestión, una caída simulada en `lab-core` no se vería, porque la gestión seguiría viva. Se registran con sus IPs de `lab-core`.

**Reservas DHCP en pfSense para los servidores.** Las IPs de `chr`, `librenms` y `glpi` no cambian y quedan anotadas en un solo lugar. Antes, LibreNMS había quedado caído porque su IP cambió. Las reservas están fuera del pool (`.100` a `.150`) y funcionan así.

**`cliente1` como plantilla.** Su snapshot `plantilla-base` no lleva la VLAN 10 persistente, para que los clones no la hereden. Al clonar: MAC nuevas, renombrar el host y regenerar el `machine-id`.

**Ya no se usa el adaptador puente.** El laboratorio no depende del router de casa. El choque de IPs entre el clon de GLPI y LibreNMS no se repite porque pfSense reparte todo.

**pfSense se administra solo por GESTION.** Se desactivó la regla oculta de antilockout y se bloquea lo que la LAN dirija a pfSense salvo ping, DNS y NTP. Los equipos de `lab-core` no pueden administrarlo; el único camino es la red de gestión, con el snapshot como respaldo si algo falla.

**DNS de pfSense: `10.0.2.3`.** Con la VPN del PC anfitrión encendida, `1.1.1.1` y `8.8.8.8` no responden desde el laboratorio, pero `10.0.2.3` (DNS interno del NAT de VirtualBox) sí. pfSense lo usa como único servidor, con reenvío activado y sin aceptar la lista que llega por DHCP en la WAN.

## 6. Convención de nombres

- Redes internas: `lab-core` y `lab-clientes`.
- VMs y hostnames: `pfsense`, `chr`, `librenms`, `glpi`, `cliente1`. Minúsculas, sin prefijo. pfSense mantiene `pfSense` como hostname interno (cosmético).
- Snapshots: `etapa-estado`. Vigentes: `diseno-listo` (cuatro VMs), `plantilla-base` (`cliente1`), `reglas-listo` (`pfsense`) y `ntp-listo` (`glpi`, `librenms` y `chr`). Los anteriores (`pfsense-inicio`, `gestion-lista`, `pfsense-base-listo`, `chr-lab-listo`, `glpi-listo`, `cliente2-plantilla-base`) se conservan por los clones enlazados.

## 7. Hora

Todos los equipos en `America/Caracas` (UTC-4).

| Equipo | Servidor NTP |
|---|---|
| pfSense | `2.pfsense.pool.ntp.org` (por defecto) |
| chr | `192.168.10.1` (pfSense) |
| glpi y librenms | `192.168.10.1` (pfSense) |
| cliente1 | El de Ubuntu por defecto |

pfSense es el servidor de hora interno de `chr`, `glpi` y `librenms` (regla UDP 123 en la LAN); `cliente1` usa el de Ubuntu por defecto. Si pfSense se apaga, esos tres pierden su fuente de hora. Todos marcan la misma hora.

GLPI: además de la zona del sistema, se fijó `date.timezone = America/Caracas` en el `php.ini` de Apache y en el de consola. Antes, `cron.log` y `event.log` marcaban UTC.

## 8. Recursos medidos

PC anfitrión: Windows, i7-2600K, 16 GB de RAM.

| Situación | Memoria | CPU |
|---|---|---|
| 4 VMs encendidas, sin ajustar RAM (con Firefox abierto) | 80 % | sin anotar |
| 5 VMs encendidas, con la RAM de la sección 4 (incluye Windows y Firefox) | 70 % | menos de 20 % |

Las 5 VMs caben a la vez. Se descarta el límite de 3. El margen, de unos 4 a 5 GB, se vuelve a medir en la etapa de incidentes (la más pesada).

## 9. Estado actual y lo que queda abierto

- Hoy `cliente1` sale a internet pasando por `chr` y pfSense. Quien enmascara hacia pfSense es el CHR: la tabla de estados de pfSense ve el tráfico de `cliente1` con origen `192.168.10.12` (verificado el 21/11/2026). O sea, hay doble NAT. La regla `masquerade` exacta del CHR está `[sin revisar en la tabla NAT actual]`. Etapa de integración: decidir dónde se hace NAT y documentarlo.
- La regla del CHR "lab no ve la casa" apunta a `192.168.1.0/24`, una red que el lab ya no usa. Revisarla o quitarla en la etapa de integración. pfSense ya bloquea `192.168.1.0/24` para toda la LAN (regla `lab no ve la casa`), así que probablemente la del CHR sobra.
- La regla `dst-nat` del puerto 2222 hacia SSH de `cliente1` nunca funcionó (causa no determinada). Con la gestión aparte ya no hace falta. Es opcional en la etapa de integración.
- LibreNMS conserva el dispositivo viejo con la IP `192.168.1.102` (marcado caído), y su `server_name` y `base_url` siguen con valores viejos. Se arregla en la etapa de monitoreo.
- pfSense se administra solo por la interfaz `GESTION` (web `https://192.168.56.11` y SSH; TCP 443 y 22 desde `192.168.56.0/24`). La regla oculta de antilockout está desactivada y la LAN no puede administrarlo. Detalle y pruebas: [pfsense/readme.md](../pfsense/readme.md).

## 10. Qué no se verificó

- Cómo la VPN del PC anfitrión bloquea `1.1.1.1` y `8.8.8.8` desde el laboratorio.
- Por qué el DNS de pfSense necesitó el modo reenvío en el primer laboratorio (no se reprodujo).
- SSH al gateway desde la LAN con el antilockout activo (solo se probó el puerto 443).
- Por qué el CHR consulta `8.8.8.8` por TCP 53.
