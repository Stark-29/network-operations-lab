# Arquitectura del Network Operations Lab

Fecha: 20/11/2026  
Actualizado: 22/11/2026  
Estado: red integrada y probada de punta a punta (NAT solo en pfSense, rutas estáticas, DNS del CHR y VLAN 10 persistente). Falta poner el monitoreo y los incidentes. Detalle de la integración: [integracion/readme.md](../integracion/readme.md).

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
                   │    pfsense    │   (el NAT se hace aquí, una sola vez)
                   └───────┬───────┘
                     em1  192.168.10.1
                           │
 ═════════ lab-core  192.168.10.0/24 ═════════════════
        │                 │                  │
   librenms .13        glpi .14        chr (ether1) .12
                                           │
                                     ┌─────┴─────┐
                                     │    chr    │   (solo enruta)
                                     └─────┬─────┘
                                  ether2  10.10.10.1
                                  vlan10  10.10.20.1
                                           │
 ═════════ lab-clientes  10.10.10.0/24 ═══════════════
                                           │
                                       cliente1
                              (DHCP del CHR: .100 a .200)
                   enp0s3.10 = VLAN 10 (10.10.20.0/24), IP 10.10.20.2
```

Rutas estáticas: pfSense, `librenms` y `glpi` envían `10.10.10.0/24` y `10.10.20.0/24` al CHR (`192.168.10.12`). Detalle en [direccionamiento](direccionamiento.md#8-rutas-estáticas).

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
| `chr` | Router interno (MikroTik CHR), DHCP de `lab-clientes` y relevo DNS | 1 GB | `lab-core`, `lab-clientes`, solo-anfitrión |
| `librenms` | Monitoreo | 2 GB | `lab-core`, solo-anfitrión |
| `glpi` | Tickets e inventario | 1 GB | `lab-core`, solo-anfitrión |
| `cliente1` | Cliente de prueba (con VLAN 10 persistente) | 1 GB | `lab-clientes`, solo-anfitrión |
| `cliente-origen` | VM de la que salió `cliente1` (clon enlazado). **No se enciende ni se borra** | n/a | n/a |

RouterOS del chr: 7.25.1 (verificado el 20/11/2026). Licencia del CHR: gratuita, con un tope de 1 Mbps de subida por interfaz (verificado el 22/11/2026; el efecto no se midió).

## 5. Decisiones y por qué

**Red de gestión aparte (fuera de banda).** Las VMs de Ubuntu no tienen copiar y pegar en la consola de VirtualBox, así que se administran por SSH desde mi PC. Esa entrada va por una red separada del tráfico que se monitorea. En un NOC real pasa igual: el canal de gestión existe para poder entrar justo cuando falla el servicio principal.

**LibreNMS monitorea por `lab-core`, no por gestión.** LibreNMS marca un equipo como caído cuando no le responde la IP registrada. Si usara la IP de gestión, una caída simulada en `lab-core` no se vería, porque la gestión seguiría viva. Se registran con sus IPs de `lab-core`.

**Reservas DHCP en pfSense para los servidores.** Las IPs de `chr`, `librenms` y `glpi` no cambian y quedan anotadas en un solo lugar. Antes, LibreNMS había quedado caído porque su IP cambió. Las reservas están fuera del pool (`.100` a `.150`) y funcionan así.

**`cliente1` como plantilla.** Su snapshot `plantilla-base` no lleva la VLAN 10 persistente, para que los clones no la hereden. `cliente1` sí la tiene (netplan) desde el 22/11/2026. Al clonar: MAC nuevas, renombrar el host y regenerar el `machine-id`.

**Ya no se usa el adaptador puente.** El laboratorio no depende del router de casa. El choque de IPs entre el clon de GLPI y LibreNMS no se repite porque pfSense reparte todo.

**NAT una sola vez, en pfSense.** El CHR enruta y no enmascara. Con doble NAT, pfSense veía todo como `192.168.10.12`; ahora ve el origen real (`10.10.10.x` y `10.10.20.x`) en estados y logs. pfSense tiene el gateway `CHR_LAB` (`192.168.10.12`) y rutas estáticas hacia las dos redes de clientes; su NAT automático ya las cubre.

**Rutas estáticas también en `glpi` y `librenms`.** Comparten segmento con el CHR. Sin rutas hacia las redes de clientes, respondían a pfSense, que descartaba las respuestas (enrutamiento asimétrico; explicación probable, no verificada en pf). Con las rutas, el tráfico entre clientes y servidores va por el CHR sin pasar por pfSense; por eso hoy el CHR, que no tiene reglas, es el único punto donde se podría filtrar.

**Reglas de la LAN con el alias `LAB_NETS`** (`lab-core`, `lab-clientes` y la VLAN 10). Un solo lugar donde se listan las redes del laboratorio. `lab no ve la casa` la aplica pfSense, con el origen real.

**DNS de `lab-clientes`: el CHR como relevo.** `cliente1` pregunta al CHR (`10.10.10.1`), el CHR a pfSense (su DNS dinámico) y pfSense a `10.0.2.3`. Evita `8.8.8.8`, que con la VPN del PC no responde. `allow-remote-requests=yes` deja el DNS del CHR abierto en todas sus interfaces.

**pfSense se administra solo por GESTION.** Se desactivó la regla oculta de antilockout y se bloquea lo que la LAN dirija a pfSense salvo ping, DNS y NTP. Los equipos de `lab-core` no pueden administrarlo; el único camino es la red de gestión, con el snapshot como respaldo si algo falla.

**DNS de pfSense: `10.0.2.3`.** Con la VPN del PC anfitrión encendida, `1.1.1.1` y `8.8.8.8` no responden desde el laboratorio, pero `10.0.2.3` (DNS interno del NAT de VirtualBox) sí. pfSense lo usa como único servidor, con reenvío activado y sin aceptar la lista que llega por DHCP en la WAN.

## 6. Convención de nombres

- Redes internas: `lab-core` y `lab-clientes`.
- VMs y hostnames: `pfsense`, `chr`, `librenms`, `glpi`, `cliente1`. Minúsculas, sin prefijo. pfSense mantiene `pfSense` como hostname interno (cosmético).
- Snapshots: `etapa-estado`. Vigente: `integrado-base` (las cinco VMs, 22/11/2026). Anteriores, que se conservan como puntos de vuelta y por los clones enlazados: `diseno-listo` (cuatro VMs), `plantilla-base` (`cliente1`, base para clones), `reglas-listo` (`pfsense`), `ntp-listo` (`glpi`, `librenms` y `chr`), `pfsense-inicio`, `gestion-lista`, `pfsense-base-listo`, `chr-lab-listo`, `glpi-listo` y `cliente2-plantilla-base`.

## 7. Hora

Todos los equipos en `America/Caracas` (UTC-4).

| Equipo | Servidor NTP |
|---|---|
| pfSense | `2.pfsense.pool.ntp.org` (por defecto) |
| chr | `192.168.10.1` (pfSense) |
| glpi y librenms | `192.168.10.1` (pfSense) |
| cliente1 | `192.168.10.1` (pfSense); de dónde sale esa configuración no está determinado |

pfSense es el servidor de hora interno de `chr`, `glpi` y `librenms` (regla UDP 123 en la LAN). `cliente1` también se sincroniza con `192.168.10.1` (verificado el 22/11/2026 con `timedatectl timesync-status`, estrato 3), aunque los documentos anteriores decían que usaba el de Ubuntu por defecto. El DHCP del CHR no entrega servidor NTP. Si pfSense se apaga, `chr`, `glpi` y `librenms` pierden su fuente de hora. Todos marcan la misma hora.

GLPI: además de la zona del sistema, se fijó `date.timezone = America/Caracas` en el `php.ini` de Apache y en el de consola. Antes, `cron.log` y `event.log` marcaban UTC.

## 8. Recursos medidos

PC anfitrión: Windows, i7-2600K, 16 GB de RAM.

| Situación | Memoria | CPU |
|---|---|---|
| 4 VMs encendidas, sin ajustar RAM (con Firefox abierto) | 80 % | sin anotar |
| 5 VMs encendidas, con la RAM de la sección 4 (incluye Windows y Firefox) | 70 % | menos de 20 % |

Las 5 VMs caben a la vez. Se descarta el límite de 3. El margen, de unos 4 a 5 GB, se vuelve a medir en la etapa de incidentes (la más pesada).

## 9. Estado actual y lo que queda abierto

- **NAT:** una sola traducción, en pfSense (decidido y verificado el 22/11/2026). El CHR no tiene reglas de NAT ni de filtro. Detalle y pruebas: [integracion/readme.md](../integracion/readme.md).
- **Filtrado:** el tráfico entre clientes y servidores pasa por el CHR sin reglas y no atraviesa pfSense. Es un límite conocido del diseño actual.
- La regla `dst-nat` del puerto 2222 hacia SSH de `cliente1` nunca funcionó (causa no determinada); se eliminó sin resolver, porque la gestión aparte ya cubre ese acceso.
- LibreNMS conserva el dispositivo viejo con la IP `192.168.1.102` (marcado caído), y su `server_name` y `base_url` siguen con valores viejos. Se arregla en la etapa de monitoreo.
- pfSense se administra solo por la interfaz `GESTION` (web `https://192.168.56.11` y SSH; TCP 443 y 22 desde `192.168.56.0/24`). La regla oculta de antilockout está desactivada y la LAN no puede administrarlo. Detalle y pruebas: [pfsense/readme.md](../pfsense/readme.md).

## 10. Qué no se verificó

- Cómo la VPN del PC anfitrión bloquea `1.1.1.1` y `8.8.8.8` desde el laboratorio.
- Por qué el DNS de pfSense necesitó el modo reenvío en el primer laboratorio (no se reprodujo).
- SSH al gateway desde la LAN con el antilockout activo (solo se probó el puerto 443).
- Por qué el CHR consultaba `8.8.8.8` por TCP 53. Hipótesis sin verificar: eran consultas de `cliente1`, vistas con origen `192.168.10.12` por el antiguo masquerade. Con el DNS nuevo, `cliente1` ya no usa `8.8.8.8`.
- De dónde sale la configuración NTP de `cliente1`.
- Por qué el primer ping de la VLAN 10, justo después de `netplan apply`, tardó 2294 ms (no se reprodujo).
- El enrutamiento asimétrico de `cliente1` hacia `glpi` y `librenms`: la explicación (pfSense descarta respuestas de conexiones que no vio empezar) es probable pero no se verificó en las reglas de pf.
