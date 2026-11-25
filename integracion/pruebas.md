# Pruebas de firewall y conectividad del Network Operations Lab

**Fecha:** 24/11/2026

Esta etapa valida la red integrada con una matriz de pruebas "esperado / real": desde `cliente1` hacia internet, el CHR, los servidores y pfSense, desde la VLAN 10 hacia la red base y la casa, y SNMP desde una IP no autorizada. También reviso el log de pfSense, los contadores del CHR, y pruebo una restauración de pfSense y los reinicios. Las pruebas encontraron dos límites del diseño (el CHR aceptaba todo de los clientes y la regla de GESTION admitía a toda su red); los corregí y repetí las pruebas afectadas.

Diseño de la red: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md). Integración: [readme.md](readme.md). pfSense: [pfsense/readme.md](../pfsense/readme.md). CHR: [mikrotik](../mikrotik/). Monitoreo: [librenms/readme.md](../librenms/readme.md). Capturas en [`capturas/pruebas/`](capturas/pruebas/).

> **Nota de transparencia:** es un laboratorio en VirtualBox, seguido con ayuda de una IA (asistente de chat). No es experiencia en producción. Lo que no pude verificar está marcado como "no verificado" o "no probado", y las causas que no determiné quedan como "causa no determinada".

## Resumen

| Tema | Resultado | Estado |
|---|---|---|
| Matriz del estado inicial | 28 filas: 27 coinciden con lo esperado y 1 coincidió en parte la primera vez (traceroute a internet); al repetirlo coincidió | Verificado; la diferencia de la primera ejecución no se reprodujo (ver [Problemas y rarezas](#problemas-y-rarezas)) |
| Hallazgo: el CHR | Los clientes llegaban a todos los servicios del CHR y al SSH de `glpi` y `librenms` | Verificado |
| Hallazgo: GESTION de pfSense | La regla admitía toda la `192.168.56.0/24`, no solo mi PC; `cliente1` llegaba a la web y al SSH de pfSense por su tarjeta de gestión | Verificado |
| Cambios | CHR: 3 reglas de `forward`, 4 servicios desactivados y 2 restringidos; pfSense: GESTION limitada a `192.168.56.1` | Verificado |
| Matriz tras los cambios | 13 filas, todas coinciden | Verificado; el rechazo de Winbox no tiene prueba directa |
| SNMP desde IP no autorizada | Sin respuesta desde `cliente1` hacia pfSense, CHR y `librenms`, con control desde `librenms` | Verificado en pfSense (log); en CHR y `librenms` la causa se deduce de la configuración |
| Log de pfSense | Las 5 pruebas de bloqueo aparecen; en la primera lectura faltaban los destinos 10.0.2.15 y 192.168.1.1, y aparecieron al repetirlas | Verificado; por qué faltaban la primera vez: causa no determinada |
| Contadores del CHR | La cola `lab-limite` contó exactamente 300 paquetes en cada sentido en una prueba de 300 pings | Verificado; la diferencia en `ether1` es hipótesis |
| Restauración de pfSense | Cambio real (regla deshabilitada) y vuelta atrás con la config vigente | Verificado por comportamiento |
| Reinicio de pfSense | 92 s hasta que volvió la web de GESTION; servicios, IPs e internet intactos | Verificado |
| Reinicio y arranque en frío del CHR | 72 s tras reiniciar y como máximo 44 s en frío; la config persistió | Verificado; el cuelgue del 23/11 no se reprodujo y su causa sigue sin determinar |

## Decisiones

| Tema | Decisión | Por qué |
|---|---|---|
| Esperado | Lo fijé antes de ejecutar cada bloque, a partir de las reglas configuradas y de los README anteriores | Una matriz solo vale si el esperado no se ajusta después a lo que salió |
| Filtrado en el CHR | Tres reglas en `forward`: aceptar respuestas, permitir a los clientes solo la web de `glpi` y descartar lo demás de clientes hacia servidores | Las pruebas B y C mostraron que los clientes llegaban al SSH de los servidores. El ciclo del proyecto es detectar, corregir y volver a probar |
| Sin regla final de descarte | Lo que no es clientes hacia servidores sigue pasando (internet, pfSense, DNS, NTP) | Con DHCP, DNS, NTP y SNMP en juego, una política de "descartar todo" sin pruebas preparadas podía cortar el laboratorio. Es un filtrado parcial, no mínimo privilegio |
| Administración del CHR | Restringir SSH y Winbox con `available-from` y desactivar `ftp`, `www`, `api` y `api-ssl`; no toqué la cadena `input` | Cubre la administración sin arriesgar DHCP, DNS ni SNMP, que no tenía pruebas preparadas |
| `available-from` y no `address` | Primero usé `address=`; RouterOS 7.25.1 lo aceptó avisando que está deprecado y lo reemplacé | Dejar documentada la sintaxis vigente |
| GESTION de pfSense | Origen `192.168.56.1` (mi PC) en las dos reglas de GESTION, en lugar de `192.168.56.0/24` | La prueba D4 mostró que cualquier VM con tarjeta de gestión llegaba a la web y al SSH |

## 1. Método

- **Orígenes:** `cliente1` (`10.10.10.200` en `lab-clientes`, `10.10.20.2` en la VLAN 10 y `192.168.56.102` en gestión) y `librenms` (`192.168.10.13`) para los controles de SNMP.
- **Herramientas:** `ping`, `nc -zv -w 3` (puertos TCP), `traceroute -n`, `curl`, `snmpget` y `ssh` con `BatchMode`. Cada bloque lleva `date +%T` al inicio y al final para casar las pruebas con el log.
- **Pruebas de bloqueo:** un *timeout* o una pérdida del 100 % no distingue "bloqueado" de "no hay nadie". Por eso cada bloqueo tiene un control: SNMP desde `librenms` (F1) y VLAN 10 hacia la casa con la regla deshabilitada ([7.1](#71-restauración-de-pfsense)).
- **Una diferencia con lo esperado no se corrige sobre la marcha:** se anota y se explica, o queda como "causa no determinada".
- **Repeticiones:** lo que salió distinto de lo esperado se repitió una vez (A2 y el log, el 24/11 a las 22:24).

**Estado de partida** (capturas 01 a 03):

- pfSense, LAN: `ping al gateway`, `dns al gateway`, `ntp al gateway`, `snmp desde librenms`, `lan no administra pfsense`, `lab no ve la casa` y `lab a internet`. Detalle en [pfsense/readme.md](../pfsense/readme.md), sección 5 ([captura 01](capturas/pruebas/01-pfsense-reglas-lan-inicio.png)).
- CHR: sin reglas de filtro ni de NAT. Servicios `ftp`, `ssh`, `www`, `winbox`, `api` y `api-ssl` activos y sin restricción de origen; `telnet` y `www-ssl` desactivados; aparece también un `reverse-proxy` en el 443 ([captura 02](capturas/pruebas/02-chr-servicios-y-reglas-inicio.png)).
- `cliente1`: ruta por defecto por `10.10.10.1` (métrica 100) y otra por `10.10.20.1` (métrica 200, solo para probar la VLAN 10). La red `192.168.56.0/24` está directa por `enp0s8`, así que llega a `192.168.56.11` y `192.168.56.12` sin pasar por ningún router ([captura 03](capturas/pruebas/03-cliente1-ip-y-rutas-inicio.png)).

## 2. Matriz del estado inicial

"Coincide": **Sí** si lo real es lo esperado; **Parcial** si coincide solo en parte.

### A. `cliente1` → internet, DNS y hora ([captura 04](capturas/pruebas/04-cliente1-internet-dns-hora.png))

| # | Prueba | Esperado | Real | Coincide |
|---|---|---|---|---|
| A1 | `ping -c 4 9.9.9.9` | 0 % de pérdida (`lab a internet`) | 4 de 4; 0 % de pérdida | Sí |
| A2 | `traceroute -n -m 4 -w 2 9.9.9.9` | Saltos `10.10.10.1`, `192.168.10.1` y `10.0.2.2`, como el 22/11 | Primera ejecución: saltos 1 y 2 como se esperaba; saltos 3 y 4 * * *. Repetido a las 22:24 (captura 32): 10.10.10.1, 192.168.10.1 y 10.0.2.2 (1,8 ms); salto 4 * * * | Parcial la primera vez; Sí al repetir |
| A3 | `resolvectl query wikipedia.org` | Resuelve (CHR → pfSense → `10.0.2.3`) | Resolvió `208.80.154.224` y `2620:0:861:ed1a::1` en 343 ms | Sí |
| A4 | `curl https://example.com` | 200 | 200 | Sí |
| A5 | `timedatectl timesync-status` | `Server: 192.168.10.1` (`ntp al gateway`) | `Server: 192.168.10.1`, estrato 2 | Sí (el 22/11 el estrato era 3) |

### B. `cliente1` → CHR ([captura 05](capturas/pruebas/05-cliente1-chr-puertos-antes.png))

| # | Prueba | Esperado | Real | Coincide |
|---|---|---|---|---|
| B1 | `ping -c 2 10.10.10.1` | 0 % (el CHR no tiene reglas) | 0 % | Sí |
| B2 | TCP 21, 22, 80, 8291, 8728 y 8729 a `10.10.10.1` | Abren: servicios activos y sin restricción | Abren los seis | Sí |
| B3 | TCP 23 a `10.10.10.1` | Rechazado: `telnet` desactivado | `Connection refused` | Sí |
| B4 | TCP 53 a `10.10.10.1` | Abre: relevo DNS (`allow-remote-requests=yes`) | Abre | Sí |
| B5 | TCP 443 a `10.10.10.1` | Por observar: `www-ssl` desactivado, pero hay un `reverse-proxy` en la lista | `Connection refused` | Sí |
| B6 | TCP 22 y 8291 a `192.168.56.12` (gestión del CHR) | Abren | Abren | Sí |

### C. `cliente1` → servidores ([captura 06](capturas/pruebas/06-cliente1-servidores-antes.png))

| # | Prueba | Esperado | Real | Coincide |
|---|---|---|---|---|
| C1 | TCP 22 y 80 a `glpi` (`192.168.10.14`) | Abren: el CHR no filtra | Abren | Sí |
| C2 | TCP 22 y 80 a `librenms` (`192.168.10.13`) | Abren | Abren | Sí |
| C3 | `traceroute -n -m 3 -w 2 192.168.10.14` | 2 saltos (`10.10.10.1` y `192.168.10.14`), sin pfSense | 2 saltos, con 0,5 ms y 1,5 ms | Sí |

### D. `cliente1` → pfSense, 11:38:51 a 11:39:19 ([captura 07](capturas/pruebas/07-cliente1-pfsense-puertos-antes.png))

| # | Prueba | Esperado | Real | Coincide |
|---|---|---|---|---|
| D1 | `ping -c 2 192.168.10.1` | Responde (`ping al gateway`) | 0 % de pérdida | Sí |
| D2 | TCP 22 y 443 a `192.168.10.1` (LAN) | *Timeout*: `lan no administra pfsense`, con el antilockout desactivado | `timed out` en los dos | Sí |
| D3 | TCP 22 y 443 a `10.0.2.15` (WAN) | *Timeout*: la misma regla (destino "This Firewall") | `timed out` en los dos | Sí; el log no muestra estos bloqueos ([sección 4](#4-log-de-pfsense)) |
| D4 | TCP 22 y 443 a `192.168.56.11` (GESTION, por la tarjeta de gestión) | Abren si la regla admite toda la `192.168.56.0/24` (hipótesis) | Abren los dos | Sí: es el hallazgo que motiva la [sección 5.2](#52-pfsense-gestion-solo-desde-mi-pc) |

### E. VLAN 10 (origen `10.10.20.2`), 11:42:27 a 11:43:10 ([captura 08](capturas/pruebas/08-vlan10-pruebas-antes.png))

| # | Prueba | Esperado | Real | Coincide |
|---|---|---|---|---|
| E1 | `ping` a `10.10.10.1` (lado `lab-clientes` del CHR) | 0 %: sin reglas en el CHR | 0 % | Sí |
| E2 | `ping` a `192.168.10.1` (pfSense) | 0 %: `10.10.20.0/24` está en `LAB_NETS` | 0 % | Sí |
| E3 | `ping` a `glpi` y `librenms` | 0 % | 0 % los dos | Sí |
| E4 | HTTP al puerto 80 de `glpi` y `librenms` | Un código HTTP | `glpi` 200; `librenms` 302 | Sí |
| E5 | HTTPS al 443 de pfSense | *Timeout*: `lan no administra pfsense` | `000` tras 5 s | Sí |
| E6 | `ping` a `192.168.1.1` (casa) | 100 % de pérdida: `lab no ve la casa` | 100 % | Sí; control en [7.1](#71-restauración-de-pfsense) |

### F. SNMP desde una IP no autorizada ([captura 09](capturas/pruebas/09-snmp-control-librenms.png) y [captura 10](capturas/pruebas/10-snmp-rechazo-cliente1.png))

`snmpget -v2c` del `sysName` (`1.3.6.1.2.1.1.5.0`). La comunidad la escribí en cada sesión sin eco y no queda en el repo.

| # | Prueba | Esperado | Real | Coincide |
|---|---|---|---|---|
| F1 | Control: desde `librenms` a `192.168.10.1`, `192.168.10.12` y `192.168.10.13` | Responden (origen autorizado) | `pfSense.lab.local`, `chr` y `librenms` | Sí |
| F2 | Desde `cliente1` a `192.168.10.1` | *Timeout*: `lan no administra pfsense` (solo `librenms` pasa por `snmp desde librenms`) | `Timeout: No Response`; 2 entradas en el log (11:49:39 y 11:49:41) | Sí |
| F3 | Desde `cliente1` a `10.10.10.1` (CHR) | *Timeout*: la comunidad solo vale desde `192.168.10.13/32` | `Timeout: No Response` | Sí |
| F4 | Desde `cliente1` a `192.168.10.13` (`librenms`) | *Timeout*: `snmpd.conf` solo acepta `192.168.10.13` y `127.0.0.1` | `Timeout: No Response` | Sí |

En F3 y F4 la causa del rechazo se deduce de la configuración: no vi un log del CHR ni de `snmpd`, y no puedo comprobar que la comunidad tecleada en las dos sesiones fuera la misma. En pfSense (F2) sí hay evidencia propia: el log.

## 3. Contadores del CHR

Esta prueba se hizo antes de crear las reglas de la [sección 5](#5-cambios-que-salieron-de-las-pruebas). Leí los contadores con `/interface print stats; /queue simple print stats` ([captura 12](capturas/pruebas/12-chr-contadores-antes.png)), lancé `ping -c 300 -i 0.2 192.168.10.14` desde `cliente1` (300 de 300 respuestas, 0 % de pérdida, 61 s, RTT mín/prom/máx de 0,548 / 1,036 / 2,197 ms) y volví a leerlos ([captura 13](capturas/pruebas/13-chr-contadores-despues.png)).

| Contador | Antes | Después | Diferencia |
|---|---|---|---|
| `ether2`, paquetes recibidos | 84 248 | 84 552 | +304 |
| `ether2`, paquetes enviados | 146 291 | 146 643 | +352 |
| `ether1`, paquetes recibidos | 156 091 | 156 773 | +682 |
| `ether1`, paquetes enviados | 93 110 | 93 816 | +706 |
| Cola `lab-limite`, paquetes (subida / bajada) | 84 063 / 147 130 | 84 363 / 147 430 | +300 / +300 |
| Cola `lab-limite`, bytes (subida / bajada) | 3 384 103 / 216 877 308 | 3 409 303 / 216 902 508 | +25 200 / +25 200 |
| Cola `lab-limite`, descartados (subida / bajada) | 0 / 332 | 0 / 332 | 0 / 0 |

- La cola contó exactamente 300 paquetes en cada sentido. Los 25 200 bytes son 300 × 84, el tamaño IP de un ping de 56 bytes de datos. Es la medida más limpia: la cola solo cuenta el tráfico de su objetivo (`10.10.10.0/24`).
- `ether2` recibió 304 paquetes (300 más 4 de otro tráfico) y envió 352 (300 más 52). Los descartes y errores de todas las interfaces no cambiaron.
- `ether1` subió más de lo que explica el ping (+682 y +706). No medí el tiempo entre las dos lecturas ni verifiqué de dónde salen los 380 a 400 paquetes sobrantes. Hipótesis, no verificada: pfSense monitorea el gateway `CHR_LAB` (`192.168.10.12`) con `dpinger`, un servicio que aparece en Status → Services.
- Sin reglas de filtro ni de NAT, el CHR no tenía contadores por regla; por eso usé los de interfaz y cola. Que este tráfico no cruza pfSense lo respaldan el traceroute C3 y las rutas, no un contador de pfSense, que no medí.

## 4. Log de pfSense

Status → System Logs → Firewall ([captura 11](capturas/pruebas/11-pfsense-log-bloqueos.png)). La pantalla decía "13 Matched Firewall Log Entries" porque tenía un filtro que mostraba solo la LAN y ocultaba IPv6 (lo puse yo para la captura):

| Hora | Origen → destino | Protocolo | Entradas | Corresponde a |
|---|---|---|---|---|
| 11:39:01 a 11:39:03 | `10.10.10.200` → `192.168.10.1:22` | TCP (SYN) | 3 | D2 |
| 11:39:04 a 11:39:06 | `10.10.10.200` → `192.168.10.1:443` | TCP (SYN) | 3 | D2 |
| 11:42:56 a 11:43:00 | `10.10.20.2` → `192.168.10.1:443` | TCP (SYN) | 5 | E5 |
| 11:49:39 y 11:49:41 | `10.10.10.200` → `192.168.10.1:161` | UDP | 2 | F2 |

| # | Entrada esperada en el log | Real | Coincide |
|---|---|---|---|
| L1 | Bloqueos de D2 (22 y 443 hacia `192.168.10.1`) | Aparecen, con 3 entradas por puerto (una por segundo) | Sí |
| L2 | Bloqueos de D3 (22 y 443 hacia `10.0.2.15`) | No aparecían en la primera lectura; al repetir la prueba (22:24) sí, con el filtro y sin él (33, 34) | No la primera vez; Sí al repetir |
| L3 | Bloqueos de E5 (443 desde la VLAN 10) | 5 entradas, una por segundo durante los 5 s de `curl -m 5` | Sí |
| L4 | Bloqueo del ping de E6 a `192.168.1.1` (`lab no ve la casa`, con registro activado) | No aparecían en la primera lectura; al repetir la prueba (22:24) sí, con el filtro y sin él (33, 34) | No la primera vez; Sí al repetir |
| L5 | Bloqueos de F2 (UDP 161 desde `cliente1`) | 2 entradas (`snmpget` con `-r 1` envía dos intentos) | Sí |

| Hora | Origen → destino | Regla | Entradas | Corresponde a |
|---|---|---|---|---|
| 22:24:36 a 22:24:38 | `10.10.10.200` → `10.0.2.15:22` (TCP, SYN) | `lan no administra pfsense` | 3 | D3 |
| 22:24:40 a 22:24:42 | `10.10.10.200` → `10.0.2.15:443` (TCP, SYN) | `lan no administra pfsense` | 3 | D3 |
| 22:24:45 y 22:24:46 | `10.10.20.2` → `192.168.1.1` (ICMP) | `lab no ve la casa` | 2 | E6 |

Repetí D3 y E6 el 24/11 a las 22:24 (la hora de `cliente1` fue de 22:24:28 a 22:24:54): los bloqueos aparecen en el log, igual con mi filtro (solo LAN, sin IPv6) y sin filtros. Por qué no estaban en la primera lectura: causa no determinada. El filtro no lo explica, porque esos paquetes entran por la LAN y son IPv4. No lo reproduje.

## 5. Cambios que salieron de las pruebas

### 5.1 CHR: filtrar clientes hacia servidores y restringir la administración

B y C mostraron que desde un cliente se llegaba a todos los servicios del CHR y al SSH de `glpi` y `librenms`. Apliqué esto en el CHR:

```
/ip firewall address-list add list=clientes address=10.10.10.0/24
/ip firewall address-list add list=clientes address=10.10.20.0/24
/ip firewall address-list add list=servidores address=192.168.10.13
/ip firewall address-list add list=servidores address=192.168.10.14
/ip firewall filter add chain=forward connection-state=established,related action=accept comment="respuestas de conexiones ya aceptadas"
/ip firewall filter add chain=forward src-address-list=clientes dst-address=192.168.10.14 protocol=tcp dst-port=80 action=accept comment="clientes a glpi web"
/ip firewall filter add chain=forward src-address-list=clientes dst-address-list=servidores action=drop comment="clientes no ven los servidores"
/ip service disable ftp,www,api,api-ssl
/ip service set ssh,winbox available-from=192.168.56.1/32
```

Estado resultante ([captura 14](capturas/pruebas/14-chr-servicios-y-reglas-nuevos.png)):

| Servicio | Puerto | Estado |
|---|---|---|
| `ftp`, `www`, `www-ssl`, `api`, `api-ssl`, `telnet` | 21, 80, 443, 8728, 8729, 23 | Desactivados (`telnet` y `www-ssl` ya lo estaban) |
| `ssh` y `winbox` | 22 y 8291 | Activos, con `available-from=192.168.56.1/32` |
| `reverse-proxy` | 443 | Figura activo y sin restricción; no acepta conexiones (B5) |

Las tres reglas de filtro quedaron en ese orden. Las cuatro entradas de las listas de direcciones (`clientes` con dos redes, `servidores` con dos IPs) son las del comando.

**Qué hace `available-from`.** Primero usé `address=`: RouterOS 7.25.1 lo aceptó avisando que está deprecado, y lo reemplacé por `available-from`. Según el manual de MikroTik, un cliente de otra dirección todavía puede abrir la conexión TCP y el servicio la cierra sin atenderla, así que un escaneo de puertos sigue viendo el puerto abierto. Lo tuve en cuenta antes de la prueba G2: para ver el rechazo real intenté una sesión SSH.

Para revertir:

```
/ip service enable ftp,www,api,api-ssl
/ip service set ssh,winbox available-from=""
/ip firewall filter remove [find where comment~"clientes|respuestas"]
/ip firewall address-list remove [find where list=clientes or list=servidores]
```

### 5.2 pfSense: GESTION solo desde mi PC

D4 mostró que `cliente1`, que tiene una tarjeta en la red de gestión, llegaba a la web y al SSH de pfSense. La causa es que las dos reglas de la pestaña GESTION (TCP 443 y TCP 22) tenían como origen toda la `192.168.56.0/24`. Cambié el origen de las dos a `192.168.56.1`, mi PC ([captura 15](capturas/pruebas/15-pfsense-reglas-gestion-apretadas.png)).

## 6. Matriz tras los cambios

Mismas pruebas que B, C, D y E, repetidas con los cambios de la sección 5.

| # | Prueba | Esperado | Real | Coincide |
|---|---|---|---|---|
| G1 | TCP 21, 23, 80, 443, 8728 y 8729 a `10.10.10.1` ([captura 16](capturas/pruebas/16-cliente1-chr-puertos-despues.png)) | Rechazados: `ftp`, `www`, `api` y `api-ssl` desactivados; `telnet` ya lo estaba; el 443 no tiene servicio que escuche | `Connection refused` en los seis | Sí |
| G2 | TCP 22 y 8291 a `10.10.10.1` | El TCP abre y el servicio cierra la sesión (ver 5.1) | Abren los dos | Sí, con el esperado corregido antes de la prueba (mi primera hipótesis era "no abren") |
| G3 | `ssh -o BatchMode=yes admin@10.10.10.1 true` | Conexión cerrada antes de autenticar | `Connection closed by 10.10.10.1 port 22` | Sí |
| G4 | TCP 22 y 8291 a `192.168.56.12`, y `ssh` con `BatchMode` | El TCP abre; el SSH se cierra antes de autenticar | Abren los dos; `Connection closed by 192.168.56.12 port 22` | Sí |
| G5 | TCP 53 a `10.10.10.1` | Sigue abierto: relevo DNS de los clientes | Abre | Sí |
| G6 | TCP 22 y 80 a `glpi` ([captura 17](capturas/pruebas/17-cliente1-servidores-despues.png)) | 22 *timeout* (`clientes no ven los servidores`); 80 abre (`clientes a glpi web`) | 22 `timed out`; 80 `succeeded` | Sí |
| G7 | TCP 22 y 80 a `librenms` | *Timeout* en los dos | `timed out` en los dos | Sí |
| G8 | `traceroute -n -m 3 -w 2 192.168.10.14` | Responde el salto 1 y los demás no: el CHR descarta la sonda nueva | `10.10.10.1`, luego `* * *` | Sí |
| G9 | VLAN 10: `ping` a `10.10.10.1` y `192.168.10.1` ([captura 18](capturas/pruebas/18-vlan10-pruebas-despues.png)) | 0 % | 0 % los dos | Sí |
| G10 | VLAN 10: `ping` a `glpi` y `librenms` | 100 %: tráfico nuevo de `clientes` hacia `servidores` | 100 % los dos | Sí |
| G11 | VLAN 10: HTTP al 80 de `glpi` y `librenms` | `glpi` responde; `librenms` *timeout* | `glpi` 200; `librenms` `000` tras 5 s | Sí |
| G12 | VLAN 10: HTTPS al 443 de pfSense y `ping` a `192.168.1.1` | *Timeout* y 100 % | `000` y 100 % | Sí |
| G13 | TCP 22 y 443 a `192.168.56.11` desde `cliente1` ([captura 19](capturas/pruebas/19-cliente1-pfsense-gestion-despues.png)) | *Timeout*: el origen ya no es `192.168.56.1` (antes abría, D4) | `timed out` en los dos | Sí |

Winbox (8291) no tiene una prueba directa de rechazo: solo vi su TCP abierto, y el rechazo real lo probé con SSH, que usa la misma opción.

## 7. Restauración y reinicios

### 7.1 Restauración de pfSense

Con la configuración vigente (ya con la GESTION limitada de la 5.2):

| Paso | Qué hice | Resultado | Captura |
|---|---|---|---|
| 1 | Respaldo en Diagnostics → Backup & Restore (`config-pfsense-restauracion.xml`, fuera del repo) | No lo capturé: contiene secretos | n/a |
| 2 | Cambio real: deshabilité `lab no ve la casa` y apliqué | Regla en gris | [20](capturas/pruebas/20-pfsense-regla-casa-deshabilitada.png) |
| 3 | Control: `ping` de la VLAN 10 a `192.168.1.1` | 0 % de pérdida: sin la regla pasa (es también el control de E6) | [21](capturas/pruebas/21-vlan10-casa-control.png) |
| 4 | Restauré el respaldo (área All) | La pantalla mostró "Package reinstall process finished successfully @ 2026-11-24 13:17:56" | [22](capturas/pruebas/22-pfsense-restauracion.png) |
| 5 | Revisé la regla en Firewall → Rules → LAN | Habilitada de nuevo | [23](capturas/pruebas/23-pfsense-regla-casa-restaurada.png) |
| 6 | `ping` VLAN 10 → casa; `ping` a `9.9.9.9`; TCP 80 a `glpi` | 100 % de pérdida; 0 %; `succeeded` | [24](capturas/pruebas/24-vlan10-casa-tras-restaurar.png) |

El mensaje de la pantalla habla de reinstalar paquetes, no dice "configuración restaurada", y no verifiqué si pfSense reinició al restaurar. La prueba es por comportamiento: la regla volvió y el ping se volvió a bloquear.

### 7.2 Reinicio de pfSense

Diagnostics → Reboot, cronometrado con `medir-arranque.ps1` sobre el puerto 443 de GESTION ([captura 25](capturas/pruebas/25-pfsense-reinicio-tiempo.png)):

| Medición | Resultado |
|---|---|
| La web dejó de responder | 13:25:44 |
| Volvió a responder | 13:27:16 |
| Tiempo sin respuesta | 92 s (el 21/11 anoté unos 1 min 40 s) |

Después, Status → Services mostraba en ejecución `bsnmpd`, `dhcpd`, `dpinger`, `ntpd`, `sshd`, `syslogd` y `unbound` ([captura 26](capturas/pruebas/26-pfsense-servicios-tras-reinicio.png)). El cliente DHCP del CHR seguía en `bound` con `192.168.10.12/24`, `librenms` conservó `192.168.10.13`, `glpi` conservó `192.168.10.14` y `cliente1` salió a internet con 0 % de pérdida ([captura 27](capturas/pruebas/27-equipos-tras-reinicio-pfsense.png)). El script mide la web de GESTION, no el plano de datos.

### 7.3 Reinicio del CHR

`/system reboot` desde SSH ([captura 28](capturas/pruebas/28-chr-reinicio-tiempo.png)): el SSH dejó de responder a las 13:33:04 y volvió a las 13:34:16, es decir, 72 s. Después ([captura 29](capturas/pruebas/29-chr-config-tras-reinicio.png)): uptime `00:02:40`; direcciones `10.10.10.1/24`, `10.10.20.1/24`, `192.168.56.12/24` y `192.168.10.12/24` (esta última dinámica, por DHCP de pfSense); las tres reglas de filtro presentes; `cliente1` con 0 % de pérdida hacia `9.9.9.9` y TCP 80 a `glpi` en `succeeded`.

### 7.4 Apagado limpio y encendido en frío del CHR

`/system shutdown`; lancé el script con la VM apagada e inicié la VM enseguida ([captura 30](capturas/pruebas/30-chr-apagado-encendido-tiempo.png)). El script arrancó a las 14:22:09 y el SSH respondió a las 14:22:52: **como máximo 44 s**, porque incluye el tiempo hasta pulsar Iniciar. No hubo cuelgue. El 23/11 el CHR se quedó en *load system* tras un apagado y encendido; en esta etapa no se reprodujo en dos arranques (un reinicio y un encendido en frío), pero no hay causa determinada. Los 72 s del reinicio y los 44 s del encendido en frío no son directamente comparables: el reinicio se mide desde que el SSH deja de responder, durante el cierre del sistema.

### 7.5 LibreNMS tras los reinicios

[Captura 31](capturas/pruebas/31-librenms-dispositivos-tras-reinicios.png), con SNMP v2c en `pfsense`, `chr` y `librenms`, y ping en `glpi`:

| Dispositivo | Uptime que muestra LibreNMS |
|---|---|
| `pfsense` | 1 h 14 min 4 s |
| `chr` | 17 min 39 s |
| `librenms` | 3 h 40 min 31 s |
| `glpi` | Sin uptime (solo ping) |

Es coherente con los reinicios: pfSense volvió a las 13:27 y el CHR arrancó a las 14:22, y los valores corresponden a una lectura hacia las 14:40.

## Problemas y rarezas

1. **Traceroute a internet con los saltos 3 y 4 sin respuesta (A2):** el 22/11 el tercer salto fue `10.0.2.2` ([integración](readme.md), sección 7). Al repetirlo el 24/11 a las 22:24 respondió (10.0.2.2, 1,8 ms) y solo el salto 4 quedó sin respuesta. Causa de la primera ejecución: no determinada; no se reprodujo.
2. **Latencia de los saltos 1 y 2 de ese mismo traceroute:** unos 58 ms cada uno, frente a 0,5 ms al mismo CHR en C3. Al repetir, entre 0,4 y 1,3 ms. No se reprodujo; causa no determinada.
3. **Log de pfSense sin las entradas de D3 y E6 en la primera lectura:** aparecieron al repetirlas [sección 4](#4-log-de-pfsense). Causa no determinada. No se reprodujo.
4. **`reverse-proxy` en el 443:** figura activo en `/ip service` pero el puerto rechaza conexiones (B5 y G1). No averigüé qué hace esa entrada en RouterOS 7.25.1 sin configurarla.
5. **Estrato NTP 2 en A5:** el 22/11 era 3. No es un fallo: depende del servidor de hora superior de pfSense, y no lo investigué.
6. **Contadores de `ether1` por encima del ping:** hipótesis del `dpinger`, no verificada ([sección 3](#3-contadores-del-chr)).
7. **Cuelgue del CHR del 23/11:** no reproducido ([7.4](#74-apagado-limpio-y-encendido-en-frío-del-chr)); causa sin determinar.

## Lo que aprendí

- Restringir un servicio no cierra el puerto: en RouterOS, `available-from` hace que el servicio rechace la sesión, pero el TCP abre. Un escaneo de puertos no prueba la restricción; hay que intentar la sesión real.
- Una regla que pasa su prueba puede ser demasiado amplia. La de GESTION admitía toda la `192.168.56.0/24` y solo se vio al probar desde un equipo que no era mi PC.
- Repetir antes de concluir: dos rarezas de la primera ejecución (el traceroute y las entradas del log) no se reprodujeron al repetirlas. Por eso las dejé como "causa no determinada" y no como hallazgos.
- Sin reglas, el CHR no tiene contadores por regla. Los de interfaz arrastran tráfico de fondo; la cola, que solo cuenta su objetivo, dio la medida limpia.
- Fijar el esperado antes de ejecutar deja ver las diferencias (como el traceroute) en lugar de justificarlas después.

## Qué no se hizo

- El CHR no tiene regla final de descarte: lo que no sea clientes hacia servidores sigue pasando. Es un filtrado parcial, no una política de mínimo privilegio.
- El rechazo de Winbox no tiene prueba directa (G2 y G4 solo muestran su TCP abierto; el rechazo real lo probé con SSH).
- Antes de los cambios no probé `ssh` con `BatchMode` al CHR, así que no hay comparación "antes" del rechazo de autenticación; solo el TCP abierto (B2 y B6).
- En SNMP, la causa del rechazo en el CHR y en `librenms` se deduce de la configuración, sin un log del equipo, y no puedo comprobar que la comunidad tecleada en las dos pruebas fuera la misma.
- Solo hay un cliente: no probé tráfico entre la VLAN 10 y otro equipo de `lab-clientes`. E1 llega solo al CHR.
- La restricción de GESTION a `192.168.56.1` la probé solo desde `cliente1`.
- No probé restaurar la configuración del CHR desde un `export` o un `backup`: solo reinicio y arranque en frío, con la configuración persistente.
- IPv6: no probé conectividad (el DNS devolvió una dirección IPv6, pero no la usé).
- No medí rendimiento, ni el efecto del tope de 1 Mbps de la licencia.
- Sin experiencia en producción con pfSense ni MikroTik.

## Referencias

- MikroTik, [Services](https://manual.mikrotik.com/docs/system-information-and-utilities/services): comportamiento de `available-from` frente a un origen no permitido.
- MikroTik, [changelog del 28/09/2026](https://manual.mikrotik.com/changelog/changelog-2026-09-28): `available-from` reemplaza al parámetro deprecado `address`.

## Versiones y herramientas

- pfSense CE 2.9.1-RELEASE (amd64)
- MikroTik RouterOS 7.25.1 (CHR)
- VirtualBox; Ubuntu Server (`librenms`, `glpi`, `cliente1`)
- `nc`, `traceroute`, `curl` y `snmpget` (net-snmp) en `cliente1` y `librenms`
- PowerShell con OpenSSH; script propio `medir-arranque.ps1` para cronometrar reinicios
