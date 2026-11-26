# Incidente 03: pfSense apagado, LibreNMS detecta la caída pero Telegram no avisa

**Fecha:** 25/11/2026 · **Ticket en GLPI:** n.º 5

Tercer incidente simulado del Network Operations Lab, y el que cerraba una duda abierta del diseño: la salida de las alertas a Telegram pasa por pfSense. Apagué pfSense para ver qué pasaba con la alerta de su propia caída. Los anteriores: [incidente-01](incidente-01.md) (CHR apagado) e [incidente-02](incidente-02.md) (DNS roto). Diseño de la red: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md); monitoreo: [librenms/readme.md](../librenms/readme.md).

> **Aviso de alcance.** Es un laboratorio en VirtualBox, no una red de producción, y el fallo lo provoqué yo: conocía la causa antes de empezar. Anoté lo que esperaba antes de apagar pfSense y diagnostiqué en orden, de lo físico a lo lógico. Seguí una guía de pasos y ayuda de una IA (asistente de chat). El ticket y el artículo son de prueba. No es experiencia en producción ni en un NOC real.

## Resumen

| Tema | Resultado |
|---|---|
| Qué hice | Apagué pfSense con Diagnostics → Halt System. Estuvo apagado 9 min 6 s (19:38:58 a 19:48:04). |
| Qué se vio | Sin pfSense, nadie tiene internet ni DNS. Entre `cliente1`, el CHR, `librenms` y `glpi` la comunicación interna siguió funcionando. |
| Cómo se detectó | LibreNMS marcó `pfsense` Down a las 19:40:14. **El aviso a Telegram no salió**: el intento falló a las 19:41:14. Lo vi en el panel de LibreNMS. |
| Causa | pfSense apagado (provocada). |
| Por qué no salió el aviso | `cURL error 28: Resolving timed out after 10000 milliseconds` al contactar `api.telegram.org`: falló la resolución del nombre. |
| Solución | Encender pfSense y comprobar. Al volver, llegó solo el aviso de recuperación. |
| Cierre | Ticket n.º 5 cerrado y artículo nuevo en la base de conocimiento. |

## Línea de tiempo

Las horas vienen del reloj de mi PC, del registro de eventos de LibreNMS y del teléfono. Todo en `America/Caracas`.

| Hora | Fuente | Qué pasó |
|---|---|---|
| 19:38:58 | PC | Hora impresa justo antes de confirmar *Halt System* (T0) |
| 19:40:14 | registro de LibreNMS | `pfsense` Down (76 s después de T0) |
| 19:40:31 a 19:40:56 | PC y `cliente1` | Revisión por capas desde el cliente |
| sin hora anotada | `cliente1` y `librenms` | Prueba al puerto 80 de `glpi` y pings desde `librenms` |
| 19:41:14 | registro de LibreNMS | Error: no se pudo emitir el aviso al transporte `telegram` |
| 19:47 | GLPI | Ticket n.º 5 abierto |
| 19:48:04 | PC | Hora impresa antes de pulsar *Start* en la VM `pfsense` |
| hacia las 19:48:38 | pfSense | Arranca (uptime de 90 s a las 19:50:08) |
| 19:49:54 a 19:50:04 | `cliente1` | Internet y DNS recuperados |
| 19:50:08 | registro de LibreNMS | `pfsense` Up; a las 19:50:09 `Device rebooted after 2 hours 25 minutes 2 seconds -> 90s` |
| 19:50:12 | mensaje de Telegram | Recuperación con `Time elapsed: 9m 58s` |
| 19:51:00 | mi teléfono | Llega ese mensaje |
| 19:51:03 | registro de LibreNMS | Aviso de recuperación emitido |
| 19:59 | GLPI | Ticket n.º 5 cerrado (hora de la última actualización que muestra el listado) |

## Lo que esperaba y lo que pasó

| Esperaba | Pasó | ¿Coincide? |
|---|---|---|
| LibreNMS marca `pfsense` Down en un chequeo de minutos terminados en 0 o 5 | 19:40:14 | Sí |
| El aviso de Telegram no sale mientras pfSense esté apagado | Error al emitir el aviso (19:41:14) y nada en el teléfono | Sí |
| `cliente1` y los servidores pierden internet | `9.9.9.9`: `Destination Host Unreachable` en `cliente1` y en `librenms` | Sí |
| `cliente1` sigue llegando a la web de `glpi` pasando solo por el CHR | `nc -zv -w 3 192.168.10.14 80`: `succeeded` | Sí |
| El DNS del cliente falla | `SERVFAIL` | Sí |
| Si el aviso de la caída aparece después, al volver pfSense | Lo ignoraba. No apareció: solo llegó la recuperación | Observado |
| Dónde fallaría el aviso | No lo había previsto: falló la resolución DNS de `api.telegram.org`, no la conexión | Dato nuevo |

## Detección

LibreNMS vio la caída en el chequeo de las 19:40:14, 76 s después de apagar pfSense. Eso es independiente de pfSense: `librenms` consulta a pfSense por `lab-core`. En el historial de alertas hay filas a las 19:40:14 de las reglas ICMP y `Device Down (SNMP unreachable)`. Lo que dependía de pfSense era el aviso: el registro de eventos de LibreNMS guardó un error a las 19:41:14, un minuto después del Down (los ~50 s habituales del motor de alertas más los 10 s de espera del intento):

```
Could not issue critical alert for rule 'Device Down! Due to no ICMP response.'
to transport 'telegram' Error: GuzzleHttp\Exception\ConnectException:
cURL error 28: Resolving timed out after 10000 milliseconds ... for
https://api.telegram.org/bot<TOKEN-TAPADO>/sendMessage?chat_id=<TAPADO>&parse_mode=HTML&text=...
```

No llegó nada al teléfono. Me enteré mirando el panel de LibreNMS, y con eso abrí el ticket.

## Diagnóstico, de lo físico a lo lógico

**Desde `cliente1`** (por su IP de gestión, solo lectura, con un script local de PowerShell, `diagnostico-cliente.ps1`; los comandos están en la columna "Prueba"). Dominio de prueba: `gnu.org`, no consultado antes.

| Capa | Prueba | Resultado | Qué dice |
|---|---|---|---|
| Enlace | `ip -br link show enp0s3` | `UP` | Bien. |
| Dirección y ruta | `ip -br addr show enp0s3` e `ip route show default` | `10.10.10.200/24` y ruta vía `10.10.10.1` | Bien. |
| Gateway | `ping -c 3 10.10.10.1` | 3 de 3 | El CHR está vivo. |
| Internet por IP | `ping -c 3 9.9.9.9` | `Destination Host Unreachable`, respondido por `10.10.10.1` | El propio CHR dice que no alcanza el siguiente salto: el problema está más allá del CHR. |
| DNS | `resolvectl query gnu.org` | `SERVFAIL` | El relevo del CHR contesta, pero no obtiene respuesta de arriba. |
| Web | `curl https://gnu.org` | código 000 | Sin salida. |

**Desde `cliente1` y `librenms` hacia dentro y hacia el borde:**

| Prueba | Resultado |
|---|---|
| `cliente1` hacia `glpi` (`192.168.10.14`, puerto 80) | `succeeded` (pasa solo por el CHR) |
| `librenms` hacia pfSense (`192.168.10.1`) | `Destination Host Unreachable`, respondido por la propia `192.168.10.13` |
| `librenms` hacia el CHR (`192.168.10.12`) | 3 de 3 (cerca de 0,7 ms) |
| `librenms` hacia `9.9.9.9` | `Destination Host Unreachable` |
| LibreNMS | `pfsense` Down desde las 19:40:14 |

**Conclusión.** Todo lo interno responde menos pfSense, que es el equipo por el que sale todo. Como `librenms` tampoco encuentra a `192.168.10.1` y el CHR sí responde, el fallo está en pfSense y no en el CHR ni en el cliente. Apagué pfSense desde su web y comprobé que la VM quedó apagada antes de tomar el snapshot `incidente-03-inicio`.

## Causa

pfSense apagado, provocado por mí. pfSense es el único camino a internet y además el DNS, el DHCP y la hora de `lab-core`.

## Por qué no salió el aviso

El error dice que la consulta DNS de `api.telegram.org` se agotó a los 10 s: falló antes de intentar conectarse. Mi explicación es que `librenms` recibe su DNS de pfSense por DHCP y que, con pfSense apagado, ni el nombre se resuelve ni habría ruta. No lo comprobé con `resolvectl` en `librenms`, así que queda como explicación probable.

Dos cosas más que observé:

- **No hubo reintento visible.** En el registro hay un solo error a las 19:41:14 y ningún intento posterior hasta la recuperación. Al volver pfSense solo llegó el aviso de recuperación; el de la caída no llegó nunca.
- **La caída se ve, pero no se avisa.** LibreNMS detecta por `lab-core`, sin pasar por pfSense; avisar sí pasa por pfSense. Si solo mirara el celular, no me habría enterado.

## Solución y verificación

1. A las 19:48:04 pulsé *Start* en la VM `pfsense`.
2. LibreNMS la vio arrancar con un uptime de 90 s a las 19:50:08, o sea que arrancó hacia las 19:48:38.
3. Revisión de `cliente1` a las 19:49:54 a 19:50:04: gateway 3 de 3, `9.9.9.9` 3 de 3 (cerca de 40 ms), `gnu.org` resuelto desde la red (159,8 ms) y web con código 200. Con eso quedó probada la cadena completa: cliente, relevo del CHR, pfSense, `10.0.2.3`.
4. LibreNMS marcó `pfsense` Up a las 19:50:08 y el aviso de recuperación salió a las 19:51:03 (llegó al teléfono a las 19:51:00).

## Ticket y artículo

- **Ticket n.º 5**, `pfSense caído: sin salida a internet`, abierto por `tecnico1` a las 19:47 a partir del panel de LibreNMS, con la nota de que la alerta de Telegram no llegó. Cerrado: el listado muestra la última actualización a las 19:59.
- Propuse la solución como `tecnico1` y la acepté como `usuario1`.
- **Artículo** en la base de conocimiento: «Si pfSense cae, la alerta de Telegram no sale». Dice que LibreNMS marca el Down en el panel pero el aviso depende de pfSense, y que ante una caída de internet sin aviso hay que mirar el panel y el estado de pfSense desde la gestión.

## Un secreto en un mensaje de error

La URL del error trae el token del bot de Telegram y el `chat_id`, y quedó guardada en el registro de eventos de LibreNMS, que es lo que se ve en pantalla y en las capturas. El token quedó expuesto en pantalla y en mis notas de trabajo, así que lo regeneré (`/revoke` en BotFather) y actualicé el transporte `telegram-lab`. En este documento y en las capturas el token y el `chat_id` están tapados. Un error de transporte puede imprimir las credenciales: hay que revisar las salidas antes de copiarlas o capturarlas.

## Lo que aprendí

- Un aviso que falla no hace ruido: el monitoreo vio la caída y el aviso no salió. El silencio no significa "todo bien", y una operación real mira también el panel.
- Detectar y avisar son dos caminos distintos. Aquí la detección no depende de pfSense; el aviso sí. Un canal de avisos que pasa por el equipo que vigila es un punto único de fallo.
- El texto del error decía dónde falló (la resolución del nombre), antes que la conexión. Conviene leerlo entero antes de suponer que "no hay internet".
- pfSense es un punto único de fallo para internet, DNS, DHCP y hora de `lab-core`. Lo interno seguía funcionando entre sí porque pasa por el CHR.
- Una idea que no probé: un canal de avisos que no dependa de pfSense, por ejemplo un camino distinto para las alertas. No sé si es viable en este laboratorio.

## Qué no se verificó

- Qué DNS usa `librenms` (se supone que pfSense, por DHCP). No lo comprobé con `resolvectl`.
- Si LibreNMS reintenta el aviso en otras condiciones. Solo vi un intento en los unos 10 min de la caída y ninguno después.
- El comando de búsqueda en `librenms.log` (con `grep` de `telegram`) no devolvió nada; la evidencia del error está en el registro de eventos de LibreNMS, no en ese archivo. Causa no determinada.
- Qué pasó con el DHCP y con la hora de `lab-core` mientras pfSense estuvo apagado. No lo probé; el CHR y `librenms` conservaron lo que ya tenían.
- La diferencia de 3 s entre la hora de llegada al teléfono (19:51:00) y la del registro (19:51:03): puede ser la diferencia entre relojes o el momento en que se registra; no la determiné.
- El historial de alertas que copié no incluye la columna de estado, así que leo las filas de las 19:50:12 como recuperaciones (interpretación). Tampoco revisé si otras reglas avisaron a Telegram; según la configuración documentada, solo la ICMP notifica.
- La hora exacta del *Halt*: uso la que imprimió el PC justo antes de confirmarlo.

## Capturas

Archivos `.png` en `capturas/`. Las capturas largas están partidas en `a` y `b`. Los nombres dicen `ticket-03` por ser el incidente 03; en GLPI es el ticket n.º 5.

[`26a-cliente1-noc-pfsense-caido`](capturas/26a-cliente1-noc-pfsense-caido.png), [`26b-cliente1-noc-pfsense-caido`](capturas/26b-cliente1-noc-pfsense-caido.png), [`27-librenms-pfsense-down`](capturas/27-librenms-pfsense-down.png), [`28-telegram-sin-aviso-pfsense`](capturas/28-telegram-sin-aviso-pfsense.png), [`29-telegram-tras-recuperar-pfsense`](capturas/29-telegram-tras-recuperar-pfsense.png), [`30-librenms-eventlog-pfsense`](capturas/30-librenms-eventlog-pfsense.png) (con el token y el `chat_id` tapados), [`31-glpi-ticket-03-cerrado`](capturas/31-glpi-ticket-03-cerrado.png), [`32-glpi-articulo-03`](capturas/32-glpi-articulo-03.png) y [`34-librenms-historial-alertas`](capturas/34-librenms-historial-alertas.png).
