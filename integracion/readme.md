# Integración de la red del Network Operations Lab

**Fecha:** 22/11/2026

Esta etapa conecta las piezas del laboratorio en una sola red: pfSense en el borde, un router MikroTik CHR detrás y los clientes detrás del CHR. Decidí dónde se hace el NAT, quién reparte DHCP y qué DNS reciben los clientes, dejé persistente la VLAN 10 y probé de punta a punta con ping, traceroute y DNS.

Diseño de la red: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md). Configuración de pfSense: [pfsense/readme.md](../pfsense/readme.md). Primer contacto con el CHR: [mikrotik](../mikrotik/). Capturas en [`capturas/`](capturas/).

> **Nota de transparencia:** es un laboratorio en VirtualBox, seguido con ayuda de una IA (asistente de chat). No es experiencia en producción. Lo que no pude verificar está marcado como "no verificado" o "no probado", y las causas que no determiné quedan como "causa no determinada".

## Topología real

```
                          Internet
                             │
                  [NAT de VirtualBox]  10.0.2.0/24  (gateway 10.0.2.2)
                             │ em0  10.0.2.15
                      ┌──────┴───────┐
                      │   pfsense    │  NAT: se hace aquí, una sola vez
                      └──────┬───────┘  rutas: 10.10.10.0/24 y 10.10.20.0/24 → 192.168.10.12
                             │ em1  192.168.10.1
 ═════════ lab-core  192.168.10.0/24 ═══════════════════════════
      │                  │                     │
 librenms .13         glpi .14           chr (ether1) .12
 (rutas hacia          (rutas hacia            │
  10.10.x.0/24          10.10.x.0/24      ether2  10.10.10.1
  vía .12)              vía .12)          vlan10  10.10.20.1
                                               │
 ═════════ lab-clientes  10.10.10.0/24 ════════╪═══════════════
                                               │
                                           cliente1  10.10.10.200 (DHCP del chr)
                                           enp0s3.10 (VLAN 10)  10.10.20.2
```

La red de gestión (solo-anfitrión, `192.168.56.0/24`) no cambia; está en [arquitectura](../docs/arquitectura.md).

## Resumen

| Tema | Resultado | Estado |
|---|---|---|
| Licencia del CHR | Gratuita: 1 Mbps de subida por interfaz | Verificado en la documentación de MikroTik; efecto no medido |
| NAT | Solo en pfSense; el CHR ya no enmascara | Verificado (estado de pfSense con el origen real) |
| DHCP | Un servidor por red: pfSense en `lab-core`, CHR en `lab-clientes` | Verificado |
| DNS de `lab-clientes` | El CHR hace de relevo y entrega `10.10.10.1` | Verificado con `cliente1` |
| Rutas | Estáticas en pfSense hacia las redes del CHR; estáticas en `glpi` y `librenms` hacia el CHR | Verificado |
| VLAN 10 | Persistente en `cliente1` (netplan); sale a internet | Verificado tras reinicio |
| Lab sin acceso a la casa | Lo bloquea pfSense con el origen real | Verificado con `cliente1`; VLAN 10 no probada |
| Enrutamiento asimétrico | Fallaba `cliente1` → `glpi`; corregido con rutas en los servidores | Explicación probable, no verificada en pf |

## Decisiones

| Tema | Decisión | Por qué |
|---|---|---|
| NAT | Una sola traducción, en pfSense; el CHR solo enruta | Con doble NAT pfSense veía todo como `192.168.10.12` y no distinguía clientes. Mi criterio: en un diseño con borde y núcleo, el borde traduce y el núcleo enruta |
| DHCP | pfSense en `lab-core` (con reservas), CHR en `lab-clientes`, VirtualBox en gestión | Un solo servidor por red; el DHCP duplicado ya fue un problema en MS Vandamme |
| DNS de los clientes | El CHR como relevo (`10.10.10.1`) | `cliente1` pregunta al CHR, el CHR a pfSense y pfSense a `10.0.2.3`; evita `8.8.8.8`, que con la VPN del PC no responde |
| Rutas | Estáticas, no dinámicas | Dos redes detrás de un solo router; no justifica OSPF |
| Reglas de la LAN | Alias `LAB_NETS` como origen | Un solo lugar donde se listan las redes del laboratorio |

## 1. Punto de partida

Antes de cambiar nada leí la configuración del CHR por SSH (solo lectura):

- Direcciones: `ether1` `192.168.10.12/24` (DHCP de pfSense, con reserva), `ether2` `10.10.10.1/24`, `vlan10` `10.10.20.1/24`, `ether3` `192.168.56.12/24` (gestión).
- Ruta por defecto: `192.168.10.1` (pfSense), aprendida por DHCP.
- NAT: `masquerade` con salida por `ether1`, sin filtro de origen, y un `dst-nat` del puerto 2222 hacia `10.10.10.200:22`.
- Filtro: una sola regla, `lab no ve la casa`, de `10.10.10.0/24` hacia `192.168.1.0/24`. No cubría la VLAN 10.
- DHCP: un servidor (`dhcp-lab`, en `ether2`, pool `10.10.10.100`-`.200`) que entregaba el DNS `8.8.8.8`. En `ether1` solo había un cliente DHCP.
- DNS: servidor dinámico `192.168.10.1` y `allow-remote-requests=no`.
- Cola: una Simple Queue, `lab-limite`, de 2M/5M sobre `10.10.10.0/24`.

Con esto quedó cerrada la marca "sin revisar" de la tabla NAT en `arquitectura.md`.

## 2. Licencia del CHR

`/system license print` muestra `level: free`. La documentación de MikroTik dice que el nivel gratuito corre sin vencimiento, con un tope de 1 Mbps de subida por interfaz y sin otras restricciones de funciones ([referencia](https://manual.mikrotik.com/docs/getting-started/routeros-licensing/chr/chr-licensing)). Existe una prueba de 60 días de los niveles de pago con cuenta de MikroTik; no la usé.

Efecto esperado: ping, traceroute, DNS y SNMP no se ven afectados. Las descargas grandes de `cliente1` probablemente topan cerca de 1 Mbps, porque salen por `ether2` (inferencia mía, no medida). La Simple Queue de 2M/5M queda por encima del tope de la licencia.

## 3. pfSense: gateway y rutas estáticas

pfSense no conocía las redes detrás del CHR. Primero añadí el CHR como gateway y después las rutas:

1. System → Routing → Gateways: `CHR_LAB`, `192.168.10.12`, interfaz LAN, descripción `chr, router interno` ([captura](capturas/01-gateway-chr-lab.png)). La pantalla no ofrece marcarlo como predeterminado.
2. System → Routing → Static Routes: `10.10.10.0/24` y `10.10.20.0/24`, ambas por `CHR_LAB` ([captura](capturas/02-rutas-estaticas.png)).
3. En el shell, `netstat -rn -f inet` muestra las dos redes por `192.168.10.12` en `em1`, con las banderas `UGS` (activa, por gateway, estática), y la ruta por defecto sigue siendo `10.0.2.2` por `em0` ([captura](capturas/03-rutas-y-nat-sin-regla.png)).

## 4. NAT: dónde se hace

Hasta aquí había doble NAT: el CHR enmascaraba y pfSense volvía a traducir. Elegí dejar el NAT solo en pfSense. El orden fue: primero pfSense, con el CHR aún enmascarando, y el masquerade al final; así `cliente1` no perdía salida mientras tanto.

El camino real:

1. `pfctl -s nat | grep "10\.10\."` salió vacío ([captura](capturas/03-rutas-y-nat-sin-regla.png)). Concluí que el NAT automático no traducía las redes de las rutas estáticas. Fue una conclusión apresurada.
2. Pasé a NAT híbrido con dos reglas manuales, una por red ([captura](capturas/04-nat-saliente-hibrido.png)). `pfctl -s nat` mostró mis dos reglas y, además, reglas automáticas que usan la tabla `<tonatsubnets>` ([captura](capturas/05-pfctl-nat-clientes.png)).
3. `pfctl -t tonatsubnets -T show` listó `10.10.10.0/24` y `10.10.20.0/24` ([captura](capturas/06-tabla-tonatsubnets.png)). El modo automático ya las cubría; mis reglas manuales eran redundantes. Coincide con la documentación de Netgate ([referencia](https://docs.netgate.com/pfsense/en/latest/nat/outbound-nat.html)).
4. Volví al modo automático y borré las dos reglas ([captura](capturas/07-nat-saliente-automatico.png), [pfctl](capturas/08-pfctl-nat-automatico.png)). La tabla conservó las dos redes.

El `grep` vacío del paso 1 no probaba nada: las subredes viven dentro de una tabla, y `grep "10\.10\."` no la ve.

## 5. Reglas de la LAN

Las reglas de la LAN tenían como origen `LAN subnets` (`192.168.10.0/24`). Con el CHR sin enmascarar, los clientes llegan con su IP real (`10.10.10.x` o `10.10.20.x`) y no coincidirían. Netgate indica que las reglas de la LAN deben permitir como origen las redes detrás de rutas estáticas ([referencia](https://docs.netgate.com/pfsense/en/latest/routing/static-routes.html)).

1. Alias `LAB_NETS` (Network(s)): `192.168.10.0/24` (`lab-core`), `10.10.10.0/24` (`lab-clientes`) y `10.10.20.0/24` (`vlan 10`) ([captura](capturas/09-alias-lab-nets.png)).
2. Cambié solo el origen de mis 5 reglas a `LAB_NETS`. Destino, puerto y log quedaron igual.
3. Añadí al final una regla Pass IPv4, origen `LAB_NETS`, destino cualquiera, `lab a internet`. La regla de fábrica de la LAN solo cubre `LAN subnets`. No probé qué pasa sin esta regla nueva; es lo que indica la documentación.

Las dos reglas de bloqueo (`lan no administra pfsense` y `lab no ve la casa`) quedan por encima del pase, así que no las tapa ([captura](capturas/10-reglas-lan-lab-nets.png), con contadores). Después del cambio, `cliente1` seguía llegando a `9.9.9.9` y `glpi` a `192.168.10.1`.

## 6. DNS de `lab-clientes`

El CHR entregaba `8.8.8.8`, que con la VPN del PC no responde desde el laboratorio. El CHR ya tenía como DNS dinámico a pfSense (`192.168.10.1`), así que lo usé de relevo:

```
/ip dns set allow-remote-requests=yes
/ip dhcp-server network set [find address=10.10.10.0/24] dns-server=10.10.10.1
```

([captura](capturas/11-chr-dns-dhcp.png)). En `cliente1`, tras `sudo networkctl renew enp0s3`, `resolvectl status` muestra `10.10.10.1` y `resolvectl query wikipedia.org` resuelve (primera consulta: 217 ms) ([captura](capturas/12-cliente1-dns.png)). La VPN del PC estaba encendida durante la prueba (en este proyecto siempre lo está).

`allow-remote-requests=yes` deja el DNS del CHR abierto en todas sus interfaces, y el CHR no tiene reglas de entrada. En el laboratorio no importa; queda en "Qué no se hizo".

## 7. Apagar el masquerade

Con pfSense listo, desactivé la regla del CHR (sin borrarla, para poder revertir):

```
/ip firewall nat disable [find action=masquerade]
```

Desde `cliente1`: `ping -c 4 9.9.9.9` respondió 4 de 4, y `traceroute -n 9.9.9.9` pasó por `10.10.10.1` (CHR), `192.168.10.1` (pfSense) y `10.0.2.2` (gateway del NAT de VirtualBox) ([captura](capturas/13-cliente1-ping-traceroute.png)). El estado de pfSense muestra `10.0.2.15 (10.10.10.200) -> 9.9.9.9`: ahora ve el origen real, no `192.168.10.12` ([captura](capturas/14-estado-nat-cliente1.png)).

## 8. Que el laboratorio no vea la casa

Hasta aquí la regla `lab no ve la casa` del CHR bloqueaba antes de que el tráfico llegara a pfSense. La desactivé para comprobar que pfSense bloquea por su cuenta. El ping de `cliente1` a `192.168.1.1` falló, y el log de pfSense muestra el bloqueo con origen `10.10.10.200` por esa regla ([captura](capturas/15-log-bloqueo-casa.png)).

## 9. Enrutamiento asimétrico

**Síntoma.** `nc -zv -w 3 192.168.10.14 80` desde `cliente1` dio *timed out* ([captura](capturas/16-cliente1-a-glpi.png)). El log de pfSense mostró bloqueos `TCP:SA` de `192.168.10.14:80` hacia `10.10.10.200`, por "Default deny" ([captura](capturas/17-log-asimetria.png)).

**Explicación (probable, no verificada en las reglas de pf).** `cliente1` llega a `glpi` por el CHR, porque están en el mismo segmento (`lab-core`). Pero `glpi` responde a su gateway, pfSense, que ve una respuesta de una conexión que nunca vio empezar y la descarta. Netgate describe el caso: un firewall con estado descarta tráfico legítimo cuando solo ve un sentido ([referencia](https://docs.netgate.com/pfsense/en/latest/routing/static-routes.html)).

**Opciones.** Netgate ofrece "Bypass firewall rules for traffic on the same interface", que añade reglas más permisivas ([referencia](https://docs.netgate.com/pfsense/en/latest/troubleshooting/asymmetric-routing.html)). No la usé: tapa el síntoma y afloja el filtrado. Preferí la causa: que `glpi` y `librenms` sepan que las redes de clientes están detrás del CHR. En ambos, un archivo netplan nuevo:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      routes:
        - to: 10.10.10.0/24
          via: 192.168.10.12
        - to: 10.10.20.0/24
          via: 192.168.10.12
```

([captura](capturas/18-glpi-rutas-netplan.png)). Tras `netplan apply`, `cliente1` llegó a `glpi` y a `librenms` por el puerto 80, y el traceroute a `glpi` fue de 2 saltos (`10.10.10.1` y `192.168.10.14`), sin pasar por pfSense ([captura](capturas/19-cliente1-a-servidores.png)).

**Consecuencia.** El tráfico entre clientes y servidores ya no pasa por pfSense. Hoy el CHR, sin reglas, es el único punto donde se podría filtrar.

## 10. VLAN 10 persistente en `cliente1`

En el primer laboratorio la VLAN 10 de `cliente1` se creó con comandos sueltos y se perdía al reiniciar. Ahora está en netplan (`/etc/netplan/60-vlan10.yaml`):

```yaml
network:
  version: 2
  vlans:
    enp0s3.10:
      id: 10
      link: enp0s3
      addresses: [10.10.20.2/24]
      routes:
        - to: default
          via: 10.10.20.1
          metric: 200
```

La ruta por defecto con métrica 200 es solo de prueba: el tráfico normal sigue por `enp0s3` (métrica 100), y con `ping -I enp0s3.10` se fuerza a usar la de la VLAN.

- Tras reiniciar, `ping 10.10.20.1` respondió 3 de 3, con 0,5 a 0,7 ms ([captura](capturas/20-vlan10-persistente.png)).
- `ping -I enp0s3.10 9.9.9.9` respondió sin pérdida, y el traceroute pasó por `10.10.20.1`, `192.168.10.1` y `10.0.2.2` ([captura](capturas/21-vlan10-internet.png)).
- El estado de pfSense muestra `10.0.2.15 (10.10.20.2) -> 9.9.9.9`: la VLAN también se traduce en pfSense ([captura](capturas/22-estado-nat-vlan10.png)).

El snapshot `plantilla-base` de `cliente1` no cambia: los clones que salgan de él no heredan la VLAN.

## 11. Limpieza del CHR

Con el NAT en pfSense, quité del CHR el masquerade, el `dst-nat` huérfano y la regla `lab no ve la casa`:

```
/ip firewall nat remove [find action=masquerade]
/ip firewall nat remove [find action=dst-nat]
/ip firewall filter remove [find comment="lab no ve la casa"]
```

([captura](capturas/23-chr-sin-nat-ni-filtro.png)). Para revertir el masquerade: `/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade`.

## 12. Snapshots

Con todo funcionando, apagué cada VM de forma limpia y tomé el snapshot `integrado-base` en las cinco: `pfsense`, `chr`, `librenms`, `glpi` y `cliente1`.

## Problemas y rarezas

1. **`pfctl` y la tabla `<tonatsubnets>`:** en el laboratorio de pfSense, `pfctl -s nat` listaba las subredes sueltas; ahora aparecen dentro de una tabla, también con el modo automático restaurado. No determiné qué cambió.
2. **Primer ping de la VLAN 10 lento:** justo después de `netplan apply`, solo 1 de 3 pings respondió y con 2294 ms. Tras reiniciar, 0,5 a 0,7 ms. Causa no determinada, no reproducida.
3. **NTP de `cliente1`:** `networkctl` muestra `NTP: 192.168.10.1`, `ip route` una ruta `192.168.10.1 via 10.10.10.1 proto dhcp` y `timedatectl timesync-status` confirma que `cliente1` se sincroniza con `192.168.10.1` (pfSense, estrato 3). Los documentos anteriores decían que `cliente1` usaba el NTP de Ubuntu por defecto, así que eso no era cierto. No determiné de dónde sale esa configuración: el DHCP del CHR tiene `ntp-server` vacío, así que no la entrega él. La sincronización funciona; probablemente la deja pasar la regla `ntp al gateway` (UDP 123, origen `LAB_NETS`), pero no lo verifiqué.
4. **`dst-nat` hacia SSH de `cliente1`:** nunca funcionó (causa no determinada, ver [mikrotik](../mikrotik/)). Lo eliminé sin resolverlo, porque la red de gestión ya cubre ese acceso.
5. **El CHR consultando `8.8.8.8` por TCP 53:** quedó sin explicar en el laboratorio de pfSense. Hipótesis sin verificar: eran consultas de `cliente1`, vistas con origen `192.168.10.12` por el masquerade. Con el DNS nuevo, `cliente1` ya no usa `8.8.8.8`.

## Lo que aprendí

- Dónde se hace el NAT cambia lo que ven los demás equipos: con NAT en pfSense, los logs y los estados muestran el origen real.
- Un firewall con estado necesita ver los dos sentidos de una conexión. Si un host comparte segmento con dos routers, tiene que saber por cuál salir a cada red.
- Una salida vacía de `grep` no prueba que algo no exista: puede estar en una tabla.
- Se cambia primero lo que puede coexistir con lo viejo (pfSense) y al final lo que corta el camino (apagar el masquerade), con la vuelta atrás a mano.
- Una prueba de bloqueo solo vale si antes se desactiva el bloqueo anterior: la regla del CHR tapaba la de pfSense.

## Qué no se hizo

- El CHR no filtra: el tráfico entre clientes y servidores pasa sin reglas, y ya no atraviesa pfSense.
- El DNS del CHR queda abierto en todas sus interfaces (`allow-remote-requests=yes`), sin reglas de entrada.
- `lab a internet` permite todo hacia fuera; no se restringió por puertos.
- IPv6: no se probó nada.
- Las rutas de `glpi` y `librenms` son manuales (netplan); no probé repartirlas por DHCP ni el "bypass" de Netgate.
- La VLAN 10 tiene un solo cliente, IP fija y sin DHCP propio.
- No medí el efecto del límite de 1 Mbps de la licencia.
- Sin experiencia en producción con pfSense ni MikroTik.

## Referencias

- MikroTik, [CHR licensing](https://manual.mikrotik.com/docs/getting-started/routeros-licensing/chr/chr-licensing).
- Netgate, [Outbound NAT](https://docs.netgate.com/pfsense/en/latest/nat/outbound-nat.html).
- Netgate, [Static Routes](https://docs.netgate.com/pfsense/en/latest/routing/static-routes.html).
- Netgate, [Troubleshooting Asymmetric Routing](https://docs.netgate.com/pfsense/en/latest/troubleshooting/asymmetric-routing.html).

## Versiones y herramientas

- pfSense CE 2.9.1-RELEASE (amd64)
- MikroTik RouterOS 7.25.1 (CHR)
- VirtualBox; Ubuntu Server (`librenms`, `glpi`, `cliente1`) con netplan
- PowerShell con OpenSSH
