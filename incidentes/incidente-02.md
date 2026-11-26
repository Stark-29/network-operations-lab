# Incidente 02: DNS roto, hay ping a una IP pero no abren las páginas

**Fecha:** 25/11/2026 · **Ticket en GLPI:** n.º 4

Segundo incidente simulado del Network Operations Lab. Desactivé el relevo DNS del router interno (MikroTik CHR) y seguí el ciclo: reporte de un usuario, ticket, diagnóstico de lo físico a lo lógico, solución, cierre y artículo. A diferencia del [incidente 01](incidente-01.md), aquí LibreNMS no detecta nada y el aviso entra por el reporte del usuario. Siguiente: [incidente-03](incidente-03.md). Diseño de la red: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md).

> **Aviso de alcance.** Es un laboratorio en VirtualBox, no una red de producción, y el fallo lo provoqué yo: conocía la causa antes de empezar. Para que el ejercicio valga algo, anoté lo que esperaba antes de provocarlo y diagnostiqué en orden, de lo físico a lo lógico, sin saltar al resultado conocido. Seguí una guía de pasos y ayuda de una IA (asistente de chat). El usuario, el ticket y el artículo son de prueba; "el usuario" es `usuario1`, que manejé yo desde una ventana de incógnito. No es experiencia en producción ni en un NOC real.

## Resumen

| Tema | Resultado |
|---|---|
| Qué hice | En el CHR: `/ip dns set allow-remote-requests=no`. El fallo estuvo activo 15 min 43 s (19:15:55 a 19:31:38). |
| Qué se vio | `cliente1` conservó gateway e internet por IP, pero no resolvía nombres: "no abre ninguna página". |
| Cómo se detectó | Por reporte del usuario (ticket abierto a las 19:17:29). LibreNMS no generó ninguna alerta. |
| Causa | El CHR tenía `allow-remote-requests=no`: no atendía las consultas DNS de los clientes. |
| Solución | `/ip dns set allow-remote-requests=yes` y comprobar desde `cliente1`. |
| Cierre | Ticket n.º 4 cerrado y artículo nuevo en la base de conocimiento. |

## Contexto del fallo

`cliente1` usa al CHR (`10.10.10.1`) como servidor DNS; el CHR lo reenvía a pfSense (su DNS dinámico, `192.168.10.1`) y pfSense a `10.0.2.3` (ver [arquitectura](../docs/arquitectura.md), sección 5). El ajuste `allow-remote-requests` es el que permite al CHR responder a otros equipos.

**Estado sano antes de provocarlo:** `/ip dns print` mostraba `allow-remote-requests: yes`, `servers` vacío y `dynamic-servers: 192.168.10.1`; `nc -zv -w 3 10.10.10.1 53` desde `cliente1` daba `succeeded`.

## Línea de tiempo

| Hora | Fuente | Qué pasó |
|---|---|---|
| 19:15:52 a 19:15:55 | PC | Ejecuto `/ip dns set allow-remote-requests=no` en el CHR (la hora de la derecha es la de después del comando) |
| 19:17:29 | GLPI | `usuario1` abre el ticket n.º 4: "No abre ninguna página web" (mi PC marcaba 19:18:06 al terminar) |
| 19:20:50 a 19:21:04 | PC y `cliente1` | Revisión por capas: todo bien hasta el DNS, que falla |
| sin hora anotada | PC | `nc` al puerto 53, `:resolve` en el CHR y `/ip dns print` |
| sin hora anotada | LibreNMS | Cuatro dispositivos Up y ninguna alerta nueva |
| 19:31:38 | PC | `/ip dns set allow-remote-requests=yes` (hora impresa antes del comando) |
| 19:32:26 a 19:32:35 | `cliente1` | Verificación: puerto 53 abierto, `kernel.org` resuelto, web 200 |
| 19:34 | GLPI | Ticket n.º 4 cerrado (hora de la última actualización que muestra el listado) |

## Lo que esperaba y lo que pasó

| Esperaba | Pasó | ¿Coincide? |
|---|---|---|
| `cliente1` conserva gateway e internet por IP | Gateway 3 de 3; `9.9.9.9` 3 de 3 (cerca de 40 ms) | Sí |
| Los nombres no resuelven | `kernel.org: resolve call failed ... Connection refused`; web con código 000 | Sí |
| El puerto 53 del CHR da *refused* | `nc`: `Connection refused` (en el estado sano daba `succeeded`) | Sí |
| El propio CHR sí resuelve nombres | `:resolve kernel.org` devolvió `172.105.4.254` | Sí |
| LibreNMS no genera alerta | Cuatro dispositivos Up y sin alertas nuevas | Sí |

Sobre la última fila: el historial de alertas de LibreNMS (copiado a las 20:02) no tiene ninguna fila entre las 18:55:11 y las 19:40:14, y el fallo duró de 19:15:55 a 19:31:38. Esa regla solo vigila el ping, y el ping a todos los equipos seguía funcionando.

## Detección

Por reporte del usuario. `usuario1` abrió el ticket 1 min 34 s después de provocarlo, con el título «No abre ninguna página web». Yo, como `tecnico1`, me asigné el ticket. No hay un chequeo que vigile la resolución de nombres: la única alerta que notifica es la del ping (ICMP).

## Diagnóstico, de lo físico a lo lógico

Revisión desde `cliente1` (por su IP de gestión, solo lectura, con un script local de PowerShell, `diagnostico-cliente.ps1`; los comandos están en la columna "Prueba"). Dominio de prueba: `kernel.org`, que no había consultado antes, para que la respuesta no saliera de la caché.

| Capa | Prueba | Resultado | Qué dice |
|---|---|---|---|
| Enlace | `ip -br link show enp0s3` | `UP` | Bien. |
| Dirección y ruta | `ip -br addr show enp0s3` e `ip route show default` | `10.10.10.200/24` y ruta vía `10.10.10.1` | Bien. |
| Gateway | `ping -c 3 10.10.10.1` | 3 de 3 (cerca de 0,5 ms) | El CHR está vivo y se llega a él. |
| Internet por IP | `ping -c 3 9.9.9.9` | 3 de 3 | Hay salida a internet. |
| DNS que usa el cliente | `resolvectl dns enp0s3` | `10.10.10.1` | Usa al CHR como DNS. |
| Consulta DNS | `resolvectl query kernel.org` | `Connection refused` | El servidor DNS está alcanzable pero rechaza la consulta. |
| Web | `curl https://kernel.org` | código 000 | Falla por el DNS. |

Hasta aquí el problema no es de red sino de resolución de nombres. Para localizarlo:

1. **¿El servidor DNS acepta conexiones?** `nc -zv -w 3 10.10.10.1 53` desde `cliente1` dio `Connection refused`. El CHR contesta, pero nada atiende en el puerto 53.
2. **¿El CHR puede resolver por sí mismo?** `:put [:resolve kernel.org]` en el CHR devolvió `172.105.4.254`. El camino del CHR hacia pfSense y hacia internet funciona; el fallo está entre el cliente y el CHR, no más arriba.
3. **¿Qué tiene configurado el CHR?** `/ip dns print` mostró `allow-remote-requests: no`, cuando antes era `yes`.
4. **¿LibreNMS lo ve?** No: cuatro dispositivos Up y sin alertas nuevas.

## Causa

`allow-remote-requests=no` en el CHR. Con ese valor el CHR resuelve para sí mismo pero no responde a las consultas de los clientes, y como `cliente1` solo conoce al CHR como DNS, deja de resolver. Lo provoqué yo.

## Solución y verificación

1. A las 19:31:38 (hora impresa antes del comando): `ssh chr "/ip dns set allow-remote-requests=yes"`. `/ip dns print` mostró `allow-remote-requests: yes`.
2. `nc -zv -w 3 10.10.10.1 53` dio `succeeded`.
3. La revisión del cliente (19:32:30 a 19:32:35) dio: gateway 3 de 3, `9.9.9.9` 3 de 3, `kernel.org` resuelto a `172.105.4.254` y a una dirección IPv6, **desde la red** (153,8 ms, no desde la caché) y web con código 200.

## Ticket y artículo

- **Ticket n.º 4**, `No abre ninguna página web`, urgencia media, solicitante `usuario1`, asignado a `tecnico1`. Abierto 19:17:29; el listado muestra la última actualización a las 19:34, cuando quedó cerrado.
- Como `tecnico1` añadí un seguimiento con el diagnóstico: el cliente conservaba red, IP y gateway; las consultas al puerto 53 se rechazaban; el CHR resolvía pero tenía `allow-remote-requests` desactivado; LibreNMS tenía los cuatro dispositivos Up y ninguna alerta relacionada.
- Propuse la solución como `tecnico1` y la acepté como `usuario1`.
- **Artículo** en la base de conocimiento: «Hay ping a una IP pero no abren las páginas (DNS)», con cinco comprobaciones (ping por IP, servidor DNS del cliente, puerto 53, resolución en el propio router y `/ip dns print`) y la nota de que LibreNMS no detecta este fallo.

## El mismo síntoma, tres mensajes distintos

Los incidentes [01](incidente-01.md), 02 y [03](incidente-03.md) tienen el mismo síntoma para el usuario ("no abre páginas") y mensajes distintos en `cliente1`:

| Incidente | Gateway (`10.10.10.1`) | Internet por IP | Mensaje del DNS | Qué indica |
|---|---|---|---|---|
| 01: CHR apagado | no responde | no | `No route to host` | El cliente no llega a su servidor DNS. |
| 02: relevo DNS desactivado | responde | sí | `Connection refused` | El servidor está alcanzable pero no atiende. |
| 03: pfSense apagado | responde | no | `SERVFAIL` | El relevo del CHR contesta, pero no obtiene respuesta de arriba. |

## Lo que aprendí

- "Hay ping a una IP pero no abren páginas" separa dos problemas: la red (si el ping por IP falla) y la resolución de nombres (si funciona).
- El texto del error orienta: *refused* es "alguien contesta pero no atiende"; *no route* es "no llego"; *SERVFAIL* es "el servidor responde pero no consigue la respuesta".
- Preguntarle al propio router (`:resolve`) parte el problema en dos: del cliente al relevo y del relevo hacia arriba.
- Un monitoreo que solo hace ping no ve este fallo. Habría que añadir una comprobación de DNS; no lo hice.
- El ajuste que hace falta para que el relevo funcione (`allow-remote-requests=yes`) es el mismo que deja el DNS del CHR abierto en todas sus interfaces (ver [arquitectura](../docs/arquitectura.md), sección 5). Es un compromiso que ya estaba anotado.
- Antes de tocar nada anoté el estado sano (`/ip dns print`), y eso hizo la solución trivial de verificar.

## Qué no se verificó

- `nc` probó el puerto 53 por TCP, pero el cliente consulta por UDP. El mensaje `Connection refused` de `resolvectl` sugiere que también se rechaza por UDP, pero no probé el puerto UDP directamente.
- Por qué RouterOS rechaza la conexión en lugar de ignorarla: lo observé, no lo investigué.
- A qué hora exacta miré LibreNMS: no la anoté. Me apoyo en que el historial de alertas no tiene filas durante todo el fallo.
- No probé qué pasa con `allow-remote-requests=no` si `cliente1` tuviera otro DNS configurado.

## Capturas

Archivos `.png` en `capturas/`. Los nombres dicen `ticket-02` por ser el incidente 02; en GLPI es el ticket n.º 4.

`18-chr-dns-antes`, `19-glpi-ticket-02-creado`, `20-cliente1-sintomas-dns-roto`, `21-chr-dns-causa`, `22-librenms-sin-alertas-dns`, `23-cliente1-dns-restaurado`, `24-glpi-ticket-02-cerrado`, `25-glpi-articulo-02` y `34-librenms-historial-alertas`.
