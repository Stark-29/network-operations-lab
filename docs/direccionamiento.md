# Direccionamiento del Network Operations Lab

Fecha: 20/11/2026  
Actualizado: 25/11/2026

Todo es de laboratorio. Las IPs son de rangos privados y las MAC son virtuales (VirtualBox); se dejan en el repo público porque pertenecen a un laboratorio local sin servicios publicados hacia internet.

---

## 1. Redes (sin choques)

| Red | Rango | Uso |
|---|---|---|
| Casa | 192.168.1.0/24 | Solo mi PC. Ninguna VM está ya en esta red |
| NAT de VirtualBox | 10.0.2.0/24 | WAN de pfSense |
| `lab-core` | 192.168.10.0/24 | Núcleo del lab |
| `lab-clientes` | 10.10.10.0/24 | Clientes detrás del CHR |
| VLAN 10 (sobre `lab-clientes`) | 10.10.20.0/24 | Pruebas de VLAN (persistente en `cliente1` desde el 22/11/2026) |
| `gestion` (solo-anfitrión) | 192.168.56.0/24 | Gestión desde mi PC |

Ningún rango se solapa con otro.

## 2. IP por equipo e interfaz

| Equipo | Interfaz | Red | IP | Cómo se asigna |
|---|---|---|---|---|
| pfsense | em0 (WAN) | NAT | 10.0.2.15/24 | DHCP de VirtualBox |
| pfsense | em1 (LAN) | `lab-core` | 192.168.10.1/24 | Estática |
| pfsense | em2 (GESTION) | gestion | 192.168.56.11/24 | Estática, sin gateway |
| chr | ether1 | `lab-core` | 192.168.10.12/24 | Reserva DHCP en pfSense |
| chr | ether2 | `lab-clientes` | 10.10.10.1/24 | Estática |
| chr | vlan10 (sobre ether2) | VLAN 10 | 10.10.20.1/24 | Estática |
| chr | ether3 | gestion | 192.168.56.12/24 | Estática, sin gateway |
| librenms | enp0s3 | `lab-core` | 192.168.10.13/24 | Reserva DHCP en pfSense |
| librenms | enp0s8 | gestion | 192.168.56.13/24 | Estática (netplan) |
| glpi | enp0s3 | `lab-core` | 192.168.10.14/24 | Reserva DHCP en pfSense |
| glpi | enp0s8 | gestion | 192.168.56.14/24 | Estática (netplan) |
| cliente1 | enp0s3 | `lab-clientes` | 10.10.10.x (se vio `.200`) | DHCP del CHR (pool .100 a .200) |
| cliente1 | enp0s3.10 | VLAN 10 | 10.10.20.2/24 | Estática (netplan, `/etc/netplan/60-vlan10.yaml`) |
| cliente1 | enp0s8 | gestion | 192.168.56.x | DHCP de VirtualBox (se vio `.101` y `.102`; puede variar) |
| PC anfitrión | adaptador solo-anfitrión | gestion | 192.168.56.1 | VirtualBox |

Mnemotecnia: el último número identifica al equipo en las dos redes (`.11` pfSense, `.12` chr, `.13` librenms, `.14` glpi).

## 3. DHCP

| Servidor | Red | Pool | Reservas |
|---|---|---|---|
| pfSense | `lab-core` | 192.168.10.100 a .150 | .12 chr, .13 librenms, .14 glpi (fuera del pool) |
| chr | `lab-clientes` | 10.10.10.100 a .200 | ninguna |
| VirtualBox | gestion | desde 192.168.56.101 | ninguna |

Un solo servidor DHCP por red. Las IPs fijas de gestión (`.11` a `.14`) quedan fuera del rango de VirtualBox. La VLAN 10 no tiene servidor DHCP: `cliente1` usa IP fija en ella.

## 4. Reservas en pfSense

| Equipo | MAC (adaptador de `lab-core`) | IP |
|---|---|---|
| chr | 08:00:27:a0:35:ab | 192.168.10.12 |
| librenms | 08:00:27:eb:99:93 | 192.168.10.13 |
| glpi | 08:00:27:61:46:0a | 192.168.10.14 |

Si se regeneran las MAC de una VM, hay que actualizar su reserva. La IP va en el campo **IP Address** de la reserva, no en *Client identifier*.

## 5. Nombres de DNS y servicios

- DNS de la LAN: pfSense (`192.168.10.1`), en modo reenvío a `10.0.2.3` (único servidor; con la VPN del PC encendida, `1.1.1.1` y `8.8.8.8` no responden desde el laboratorio).
- DNS que el CHR entrega a `lab-clientes`: `10.10.10.1` (el propio CHR, como relevo). El CHR obtiene su DNS de pfSense (`192.168.10.1`) por DHCP y tiene `allow-remote-requests=yes`.
- Hora de la LAN: pfSense (`192.168.10.1`, UDP 123) para `chr`, `glpi` y `librenms`. `cliente1` también se sincroniza con `192.168.10.1`, que le llega en el lease DHCP del CHR (ver [arquitectura](arquitectura.md#7-hora)).

## 6. Acceso desde mi PC

| Servicio | Dirección |
|---|---|
| Web de pfSense | `https://192.168.56.11` (solo desde 192.168.56.1) (certificado autofirmado) |
| Winbox o SSH al CHR | `192.168.56.12` (solo desde 192.168.56.1) |
| LibreNMS | `http://192.168.56.13` |
| GLPI | `http://192.168.56.14` |
| SSH a las Ubuntu | `ssh robert@192.168.56.13` y `.14`; `cliente1` por su IP de gestión |
| SSH a pfSense | `ssh admin@192.168.56.11`(solo desde 192.168.56.1) |

En mi PC, el archivo `.ssh/config` define alias para entrar por GESTION: `ssh pfsense`, `ssh chr`, `ssh libre`, `ssh glpi` y `ssh cliente`. Ese archivo es local y no está en el repo.

## 7. Rutas por defecto

Las IPs de gestión no llevan gateway ni ruta por defecto (en Ubuntu, `use-routes: false` o interfaz sin gateway), para que el tráfico a internet siga yendo por `lab-core`.

## 8. Rutas estáticas

| Equipo | Destino | Por | Dónde se configura |
|---|---|---|---|
| pfsense | 10.10.10.0/24 y 10.10.20.0/24 | `192.168.10.12` (gateway `CHR_LAB`) | System → Routing |
| glpi y librenms | 10.10.10.0/24 y 10.10.20.0/24 | `192.168.10.12` | `/etc/netplan/60-rutas-lab.yaml` |
| chr | 0.0.0.0/0 | `192.168.10.1` (pfSense) | DHCP de pfSense |
| cliente1 | 0.0.0.0/0 | `10.10.10.1` (chr) | DHCP del CHR |
| cliente1 | 0.0.0.0/0, métrica 200, por `enp0s3.10` | `10.10.20.1` (chr) | netplan; solo para probar la VLAN 10 con `ping -I` |

Sin las rutas de `glpi` y `librenms`, sus respuestas a los clientes iban a pfSense y se descartaban (enrutamiento asimétrico; detalle en [integracion/readme.md](../integracion/readme.md)).

## 9. Alias y NAT en pfSense

- Alias `LAB_NETS`: `192.168.10.0/24` (`lab-core`), `10.10.10.0/24` (`lab-clientes`) y `10.10.20.0/24` (`vlan 10`). Es el origen de las reglas de la LAN.
- NAT saliente: automático, solo en pfSense; traduce a `10.0.2.15`. El CHR no enmascara.

## 10. Monitoreo (LibreNMS)

`librenms` (`192.168.10.13`) registra los equipos por sus IPs de `lab-core`. Detalle y pruebas: [librenms/readme.md](../librenms/readme.md).

| Equipo | IP registrada | Método | Quién puede consultar SNMP |
|---|---|---|---|
| pfsense | 192.168.10.1 | SNMP v2c y ping | Solo `192.168.10.13`: regla `snmp desde librenms` (UDP 161) y servicio atado a la LAN |
| chr | 192.168.10.12 | SNMP v2c y ping | Solo `192.168.10.13/32` (direcciones de la comunidad) |
| librenms | 192.168.10.13 | SNMP v2c y ping | `192.168.10.13` y `127.0.0.1` (`snmpd.conf`) |
| glpi | 192.168.10.14 | Solo ping | Sin SNMP |

- La comunidad SNMP es una sola para los tres equipos y no está en el repo.
- Regla de pfSense (pestaña LAN): Pass, UDP, origen `192.168.10.13`, destino LAN address, puerto 161, descripción `snmp desde librenms`. Lo demás que la LAN dirija a pfSense sigue bloqueado, salvo ping, DNS y NTP.
- Dirección web de LibreNMS: `http://192.168.56.13`. `server_name` de Nginx: `192.168.56.13 192.168.10.13`. `base_url` y `APP_URL`: `http://192.168.56.13`.
- Alertas: `librenms` sale a `api.telegram.org` (HTTPS) por pfSense, con la regla `lab a internet` y el NAT de pfSense (`10.0.2.15`). La salida a Telegram depende de pfSense (con pfSense apagado el aviso no sale; probado el 25/11/2026).
- Ya no se monitorea `192.168.1.102` ni el router de casa (la red del laboratorio no ve la red de casa).

## 11. Filtrado en el CHR

- Listas clientes (10.10.10.0/24 y 10.10.20.0/24) y servidores (192.168.10.13 y 192.168.10.14).
- Las 3 reglas de forward, en orden.
- Los servicios ssh y winbox restringidos a 192.168.56.1/32.
