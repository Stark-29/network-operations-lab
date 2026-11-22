# Direccionamiento del Network Operations Lab

Fecha: 20/11/2026

Todo es de laboratorio. Las IPs y las MAC son virtuales; decidir en el chat 8 si se dejan en el repo público.

---

## 1. Redes (sin choques)

| Red | Rango | Uso |
|---|---|---|
| Casa | 192.168.1.0/24 | Solo mi PC. Ninguna VM está ya en esta red |
| NAT de VirtualBox | 10.0.2.0/24 | WAN de pfSense |
| `lab-core` | 192.168.10.0/24 | Núcleo del lab |
| `lab-clientes` | 10.10.10.0/24 | Clientes detrás del CHR |
| VLAN 10 (sobre `lab-clientes`) | 10.10.20.0/24 | Pruebas de VLAN (persistente en el chat 4) |
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
| cliente1 | enp0s3 | `lab-clientes` | 10.10.10.x | DHCP del CHR (pool .100 a .200) |
| cliente1 | enp0s8 | gestion | 192.168.56.x | DHCP de VirtualBox (se vio `.101`; puede variar) |
| PC anfitrión | adaptador solo-anfitrión | gestion | 192.168.56.1 | VirtualBox |

Mnemotecnia: el último número identifica al equipo en las dos redes (`.11` pfSense, `.12` chr, `.13` librenms, `.14` glpi).

## 3. DHCP

| Servidor | Red | Pool | Reservas |
|---|---|---|---|
| pfSense | `lab-core` | 192.168.10.100 a .150 | .12 chr, .13 librenms, .14 glpi (fuera del pool) |
| chr | `lab-clientes` | 10.10.10.100 a .200 | ninguna |
| VirtualBox | gestion | desde 192.168.56.101 | ninguna |

Un solo servidor DHCP por red. Las IPs fijas de gestión (`.11` a `.14`) quedan fuera del rango de VirtualBox.

## 4. Reservas en pfSense

| Equipo | MAC (adaptador de `lab-core`) | IP |
|---|---|---|
| chr | 08:00:27:a0:35:ab | 192.168.10.12 |
| librenms | 08:00:27:eb:99:93 | 192.168.10.13 |
| glpi | 08:00:27:61:46:0a | 192.168.10.14 |

Si se regeneran las MAC de una VM, hay que actualizar su reserva. La IP va en el campo **IP Address** de la reserva, no en *Client identifier*.

## 5. Nombres de DNS y servicios

- DNS de la LAN: pfSense (`192.168.10.1`), en modo reenvío a `10.0.2.3` (único servidor; con la VPN del PC encendida, `1.1.1.1` y `8.8.8.8` no responden desde el laboratorio).
- Hora de la LAN: pfSense (`192.168.10.1`, UDP 123) para `chr`, `glpi` y `librenms`.
- DNS que el CHR entrega a `lab-clientes`: `8.8.8.8` (del lab de MikroTik) `[revisar en el chat 4]`.

## 6. Acceso desde mi PC

| Servicio | Dirección |
|---|---|
| Web de pfSense | `https://192.168.56.11` (certificado autofirmado) |
| Winbox o SSH al CHR | `192.168.56.12` |
| LibreNMS | `http://192.168.56.13` |
| GLPI | `http://192.168.56.14` |
| SSH a las Ubuntu | `ssh robert@192.168.56.13` y `.14`; `cliente1` por su IP de gestión |
| SSH a pfSense | `ssh admin@192.168.56.11` |

## 7. Rutas por defecto

Las IPs de gestión no llevan gateway ni ruta por defecto (en Ubuntu, `use-routes: false` o interfaz sin gateway), para que el tráfico a internet siga yendo por `lab-core`.
