# Incidente 01: CHR apagado, los clientes pierden internet

**Fecha:** 25/11/2026 · **Ticket en GLPI:** n.º 3

Primer incidente simulado del Network Operations Lab. Apagué el router interno (MikroTik CHR) con LibreNMS y GLPI en marcha y recorrí el ciclo de un NOC (centro de operaciones de red): detectar, abrir el ticket, diagnosticar de lo físico a lo lógico, resolver, cerrar y documentar. Los otros dos incidentes: [incidente-02](incidente-02.md) (DNS roto) e [incidente-03](incidente-03.md) (pfSense apagado). Diseño de la red: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md).

> **Aviso de alcance.** Es un laboratorio en VirtualBox, no una red de producción, y el fallo lo provoqué yo: conocía la causa antes de empezar. Para que el ejercicio valga algo, anoté lo que esperaba antes de ejecutar cada prueba y diagnostiqué en orden, de lo físico a lo lógico, sin saltar al resultado conocido. Seguí una guía de pasos y ayuda de una IA (asistente de chat). Los usuarios, los tickets y los artículos son de prueba. No es experiencia en producción ni en un NOC real.

## Resumen

| Tema | Resultado |
|---|---|
| Qué hice | Apagué el CHR con `/system shutdown`. Estuvo apagado 39 min 29 s (18:05:38 a 18:45:07). |
| Qué se vio | `cliente1`, detrás del CHR, perdió el gateway, internet y DNS. `librenms`, `glpi` y `pfsense` siguieron funcionando entre sí y con internet. |
| Cómo se detectó | LibreNMS marcó `chr` Down a las 18:10:13 y el aviso de Telegram salió a las 18:11:02. |
| Causa | CHR apagado (provocada). En VirtualBox la VM estaba en *Powered Off*. |
| Solución | Encender la VM y comprobar desde `cliente1` y desde `librenms`. |
| Cierre | Ticket n.º 3 cerrado y artículo nuevo en la base de conocimiento. |

## Entorno y mediciones previas

- **VMs:** las cinco encendidas (`pfsense`, `chr`, `librenms`, `glpi`, `cliente1`), con la red integrada y probada de las etapas anteriores.
- **Snapshots de vuelta:** `incidentes-inicio` en las cinco VMs, con todas apagadas. Mientras el CHR estaba apagado en este incidente tomé `incidente-02-inicio` en `chr`: su disco ya era el sano y no añadía tiempo de caída.
- **Hora:** antes de empezar comparé el reloj de los equipos. El PC, `librenms`, `glpi`, `cliente1` y `chr` marcaron la misma hora al minuto (17:56:53 a 17:57:03, `America/Caracas`). Los segundos de diferencia vienen de ejecutar los comandos uno tras otro; no medí deriva. pfSense mostró 17:58:37 en su panel con mi PC en las 17:58. GLPI también muestra hora de Caracas y no UTC: el ticket n.º 3 figura abierto a las 18:38:21 y mi PC marcaba 18:39:34 al terminar de guardarlo.
- **RAM del PC anfitrión** (cinco VMs, Firefox y Telegram abiertos): 75 % al empezar (11,9 de 15,9 GB usados, 4,0 libres) y 81 % al terminar, a las 20:02 (12,8 GB usados, 3,1 libres). No averigüé a qué se debe el aumento porque no medí por proceso. La CPU no la anoté.
- **Estado sano antes de provocar nada** (18:00): `cliente1` con gateway 3 de 3, `9.9.9.9` 3 de 3, `wikipedia.org` resuelto (desde la caché de `systemd-resolved`) y web con código 200. LibreNMS con los cuatro dispositivos Up.

## Línea de tiempo

Las horas vienen del reloj de mi PC, del reloj del CHR y del registro de eventos de LibreNMS. Todo en `America/Caracas`.

| Hora | Fuente | Qué pasó |
|---|---|---|
| 18:05:38 | reloj del CHR | Hora anotada justo antes de `/system shutdown` (T0) |
| 18:06:44 a 18:07:11 | PC y `cliente1` | Revisión por capas desde el cliente: sin gateway ni internet |
| 18:09:07 | `librenms` | Llega a pfSense e internet, no al CHR |
| 18:10:13 | registro de LibreNMS | `chr` Down (chequeo ICMP y SNMP); es la hora que lleva el mensaje de Telegram |
| 18:11:02 | registro de LibreNMS | Aviso emitido al transporte `telegram` (49 s después) |
| sin hora anotada | VirtualBox | Snapshot `incidente-02-inicio` del `chr` apagado |
| 18:38:21 | GLPI | Ticket n.º 3 abierto |
| 18:45:07 | PC | *Start* en la VM `chr` |
| 18:46:39 | CHR | Responde por SSH con uptime de 1 min 7 s (arrancó hacia las 18:45:32) |
| 18:48:56 a 18:49:05 | `cliente1` y `librenms` | Verificación: gateway 3 de 3, internet 3 de 3, web 200; `librenms` llega al CHR |
| 18:50:07 | registro de LibreNMS | `chr` Up |
| 18:50:11 | mensaje de Telegram | Recuperación con `Time elapsed: 39m 58s` |
| 18:51:02 | registro de LibreNMS | Aviso de recuperación emitido |
| 18:58 | GLPI | Ticket n.º 3 cerrado (hora de la última actualización que muestra el listado) |

## Lo que esperaba y lo que pasó

Escribí la columna de la izquierda antes de apagar el CHR.

| Esperaba | Pasó | ¿Coincide? |
|---|---|---|
| `cliente1` pierde gateway, internet por IP y DNS | Gateway 0 de 3, `9.9.9.9` 0 de 3, DNS con `No route to host`, web con código 000 | Sí |
| El enlace de `cliente1` sigue en UP | `enp0s3 UP` | Sí |
| `librenms` sigue llegando a pfSense y a internet | 3 de 3 y 3 de 3 | Sí |
| La gestión del CHR (`192.168.56.12`) deja de responder | Ping sin respuesta y SSH con *Connection timed out* | Sí |
| LibreNMS marca Down en un chequeo de minutos terminados en 0 o 5 | 18:10:13 | Sí |
| Telegram avisa unos 50 s después | 18:11:02 (49 s) | Sí |
| La alerta llega en menos de unos 6 min desde el apagado | 5 min 24 s | Sí |
| La línea ARP del cliente muestra `FAILED` o `INCOMPLETE` | Mostró `DELAY` | No |
| La consulta DNS tarda en fallar y alarga la revisión (hasta cerca de 1 min) | El script entero tardó 23 s y el DNS falló al instante con `No route to host` | No |

Sobre el `DELAY`: salió igual en revisiones con el gateway sano (por ejemplo, a las 18:48:59), así que esa línea no distingue por sí sola un gateway caído de uno sano. La evidencia fue el ping. Mi explicación probable es que la tabla de vecinos de Linux estaba en un estado intermedio al leerla, a los pocos segundos de empezar; no repetí la lectura para comprobarlo.

## Detección

LibreNMS comprueba el ICMP cada 5 minutos, en los segundos 7 a 13 de los minutos terminados en 0 o 5 (detalle en [librenms/readme.md](../librenms/readme.md), sección 6). Apagué el CHR a las 18:05:38, 25 s después del chequeo de las 18:05:13, así que el que lo vio fue el de las 18:10:13: 4 min 35 s después del apagado. El aviso salió 49 s más tarde (18:11:02), 5 min 24 s después del apagado, dentro del peor caso de unos 6 minutos que había estimado.

El mensaje de Telegram trae la regla (`Device Down! Due to no ICMP response.`), la severidad crítica, la hora, la IP del equipo (`192.168.10.12`) y datos del dispositivo (`sysDescr = RouterOS CHR 7.25.1`). Con ese aviso abrí el ticket.

## Diagnóstico, de lo físico a lo lógico

**Desde `cliente1`** (por su IP de gestión, con una revisión de solo lectura en seis capas que hice con un script local de PowerShell, `diagnostico-cliente.ps1`; los comandos están en la columna "Prueba"):

| Capa | Prueba | Resultado | Qué dice |
|---|---|---|---|
| Enlace | `ip -br link show enp0s3` | `UP` | El enlace virtual sigue arriba aunque el otro extremo esté apagado. No descarta nada. |
| Dirección y ruta | `ip -br addr show enp0s3` e `ip route show default` | `10.10.10.200/24` y ruta por defecto vía `10.10.10.1` | El cliente conserva su configuración. |
| Gateway | `ping -c 3 10.10.10.1` | 0 de 3 | El primer salto no responde. |
| Vecinos (ARP) | `ip neigh show 10.10.10.1` | `DELAY` | No concluyente (ver arriba). |
| Internet por IP | `ping -c 3 9.9.9.9` | 0 de 3 | No sale. |
| DNS | `resolvectl query debian.org` | `No route to host` | El cliente ni siquiera llega a su servidor DNS (`10.10.10.1`). |
| Web | `curl https://debian.org` | código 000 | Sin conexión. |

**Desde el lado del NOC:**

| Prueba | Resultado |
|---|---|
| LibreNMS, lista de dispositivos | `chr` Down; `pfsense`, `librenms` y `glpi` Up |
| `librenms` hacia el CHR (`192.168.10.12`) | 0 de 3 |
| `librenms` hacia pfSense (`192.168.10.1`) | 3 de 3 (cerca de 0,7 ms) |
| `librenms` hacia `9.9.9.9` | 3 de 3 (cerca de 38 ms) |
| Mi PC hacia la gestión del CHR (`192.168.56.12`) | Sin respuesta; Windows contestó "Host de destino inaccesible" desde `192.168.56.1` |
| SSH al CHR | `Connection timed out` |
| VirtualBox | `chr` en *Powered Off*; las otras cuatro VMs en ejecución |

**Conclusión.** El CHR no respondía por ninguna de sus tres redes (`lab-clientes` desde `cliente1`, `lab-core` desde `librenms` y gestión desde mi PC), mientras el resto de `lab-core` iba bien. Eso apunta al equipo entero y no a un servicio o a una interfaz. La VM en *Powered Off* lo confirmó. La configuración del cliente estaba bien: el fallo estaba en el primer salto.

## Causa

CHR apagado. La provoqué yo con `/system shutdown`.

## Solución y verificación

1. Encendí la VM `chr` desde VirtualBox a las 18:45:07.
2. A las 18:46:39 respondió por SSH con un uptime de 1 min 7 s. Arrancó sin colgarse.
3. Comprobé desde `cliente1` (18:48:59 a 18:49:05): gateway 3 de 3, `9.9.9.9` 3 de 3 (cerca de 40 ms), web con código 200. Desde `librenms`, el CHR respondió 3 de 3.
4. LibreNMS lo vio Up a las 18:50:07 y el aviso de recuperación salió a las 18:51:02. Entre que el CHR arrancó (hacia las 18:45:32) y que LibreNMS lo marcó Up pasaron unos 4 min 35 s, porque el chequeo de las 18:45 ocurrió antes de que terminara de arrancar.

En esa verificación el DNS de `debian.org` se resolvió desde la caché de `systemd-resolved` (2,1 ms), así que esa revisión por sí sola no prueba que el relevo DNS del CHR funcionara de nuevo. No anoté ninguna otra consulta de ese nombre entre las 18:07 y las 18:48; el origen de esa entrada de caché es una causa no determinada. El DNS sí quedó probado "desde la red" en los incidentes 02 y 03.

## Ticket y artículo

- **Ticket n.º 3**, `CHR caído: clientes de lab-clientes sin internet`, tipo incidencia, urgencia alta, solicitante `usuario1`, asignado a `tecnico1`. Lo abrí a las 18:38:21 pegando el texto de la alerta.
- Solo añadí el seguimiento de la causa ("CHR apagado"); el diagnóstico por capas está en este documento.
- Propuse la solución como `tecnico1`, la acepté como `usuario1` (ventana de incógnito) y el ticket quedó cerrado; el listado muestra la última actualización a las 18:58.
- **Artículo** en la base de conocimiento: «Clientes sin internet: revisar de lo físico a lo lógico». Es una lista de seis comprobaciones (físico, enlace, dirección y ruta, gateway, internet por IP y DNS) con el caso de este incidente.

El ticket se abrió 27 min después del aviso (18:11:02 a 18:38:21). En el ensayo me detuve a tomar capturas y el snapshot de `chr` y no anoté el desglose. En un NOC real el ticket se abre al recibir la alerta.

## Lo que aprendí

- En VirtualBox, un enlace en UP no prueba que el equipo de enfrente esté vivo: el cliente seguía con el enlace arriba y con su ruta por defecto. No puedo decir qué vería con equipos físicos.
- Si un equipo no responde por ninguna de sus redes y los demás siguen bien, se sospecha del equipo y no de una interfaz.
- El panel de monitoreo va con retraso en los dos sentidos: vio la caída 4 min 35 s después de apagar el CHR y vio el Up unos 4 min 35 s después de que arrancara. Antes de cerrar el ticket comprobé a mano desde el cliente.
- Escribir lo esperado antes de probar muestra dónde me equivoco: aquí fue la lectura del ARP.
- El mismo síntoma ("no abre páginas") tiene mensajes distintos según la causa; los comparo en el [incidente 02](incidente-02.md).

## Qué no se verificó

- Por qué el ARP mostró `DELAY` (la explicación de arriba es probable, no comprobada).
- El origen de la entrada de caché de `debian.org` en la verificación.
- La hora a la que el mensaje llegó al teléfono: no la anoté; uso la del registro de LibreNMS.
- Si las otras reglas (`Device Down (SNMP unreachable)` y `Device rebooted`) avisaron a Telegram. El historial de alertas tiene filas de esas reglas a las 18:50:11 y 18:55:11, pero no revisé el chat. Según la configuración documentada, solo la regla ICMP notifica.
- El cuelgue del CHR al arrancar del 23/11/2026 no se reprodujo: el CHR arrancó dos veces el 25/11/2026 (al encender las VMs y en este incidente) sin colgarse. La causa de aquel cuelgue sigue sin determinar.
- Cuánto del retraso de 27 min hasta el ticket fue el snapshot y cuánto las capturas.

## Capturas

Archivos `.png` en `capturas/`. Las capturas largas están partidas en `a` y `b`. Los nombres dicen `ticket-01` por ser el incidente 01; en GLPI es el ticket n.º 3.

- **Antes:** [`01-snapshots-incidentes-inicio`](capturas/01-snapshots-incidentes-inicio.png), [`02-hora-y-ram-antes`](capturas/02-hora-y-ram-antes.png), [`03-pfsense-hora-dashboard`](capturas/03-pfsense-hora-dashboard.png), [`04-cliente1-diagnostico-sano`](capturas/04-cliente1-diagnostico-sano.png), [`05-librenms-dispositivos-antes`](capturas/05-librenms-dispositivos-antes.png).
- **Durante:** [`06-cliente1-sintomas-chr-caido`](capturas/06-cliente1-sintomas-chr-caido.png), [`07-noc-sintomas-chr-caido`](capturas/07-noc-sintomas-chr-caido.png), [`08-telegram-alerta-chr-down`](capturas/08-telegram-alerta-chr-down.png), [`09-librenms-dispositivos-chr-down`](capturas/09-librenms-dispositivos-chr-down.png), [`10-librenms-eventlog-chr-down`](capturas/10-librenms-eventlog-chr-down.png), [`11-glpi-ticket-01-creado`](capturas/11-glpi-ticket-01-creado.png), [`12-virtualbox-chr-apagado`](capturas/12-virtualbox-chr-apagado.png).
- **Después:** [`13a-cliente1-tras-solucion-chr`](capturas/13a-cliente1-tras-solucion-chr.png), [`13b-cliente1-tras-solucion-chr`](capturas/13b-cliente1-tras-solucion-chr.png), [`14-telegram-recuperacion-chr`](capturas/14-telegram-recuperacion-chr.png), [`15-librenms-eventlog-chr-up`](capturas/15-librenms-eventlog-chr-up.png), [`16a-glpi-ticket-01-cerrado`](capturas/16a-glpi-ticket-01-cerrado.png), [`16b-glpi-ticket-01-cerrado`](capturas/16b-glpi-ticket-01-cerrado.png), [`17-glpi-articulo-01`](capturas/17-glpi-articulo-01.png), [`33-hora-y-ram-despues`](capturas/33-hora-y-ram-despues.png), [`34-librenms-historial-alertas`](capturas/34-librenms-historial-alertas.png).
