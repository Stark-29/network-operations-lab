# Incidentes simulados del Network Operations Lab

Tres incidentes provocados el 25/11/2026 sobre la red ya integrada y monitoreada ([arquitectura](../docs/arquitectura.md), [integración](../integracion/readme.md)). En cada uno recorrí el ciclo de un NOC (centro de operaciones de red): **detectar, abrir el ticket, diagnosticar de lo físico a lo lógico, resolver, cerrar y documentar**. Cada documento trae su línea de tiempo con horas, la comparación entre lo que esperaba y lo que pasó, el ticket de GLPI y el artículo de la base de conocimiento.

> **Aviso de alcance.** Es un laboratorio en VirtualBox, no una red de producción, y los fallos los provoqué yo: conocía la causa antes de empezar. Para que el ejercicio valga algo, anoté lo que esperaba antes de ejecutar cada prueba y diagnostiqué en orden, sin saltar al resultado conocido. Seguí una guía de pasos y ayuda de una IA (asistente de chat). Los usuarios, los tickets y los artículos son de prueba. No es experiencia en producción ni en un NOC real.

## Los tres incidentes

| # | Documento | Qué provoqué | Duración | Cómo se detectó | Ticket en GLPI |
|---|---|---|---|---|---|
| 01 | [incidente-01.md](incidente-01.md) | CHR apagado (`/system shutdown`) | 39 min 29 s | LibreNMS marcó Down a las 18:10:13 y el aviso de Telegram salió a las 18:11:02 (49 s después) | n.º 3 |
| 02 | [incidente-02.md](incidente-02.md) | `allow-remote-requests=no` en el CHR (DNS roto) | 15 min 43 s | Por reporte de `usuario1`; LibreNMS no alertó porque solo vigila ping | n.º 4 |
| 03 | [incidente-03.md](incidente-03.md) | pfSense apagado | 9 min 6 s | LibreNMS marcó Down a las 19:40:14, pero **el aviso a Telegram no salió** | n.º 5 |

El ticket n.º 1 es del primer laboratorio de GLPI ([glpi](../glpi/readme.md)) y el n.º 2 no corresponde a ningún ejercicio documentado. Los nombres de las capturas dicen `ticket-01`, `ticket-02` y `ticket-03` por el número de incidente; en GLPI son los tickets n.º 3, 4 y 5.

## Método

1. **Estado sano primero.** Antes de provocar nada anoté el estado normal (por ejemplo `/ip dns print` en el incidente 02) y comprobé que los cuatro dispositivos de LibreNMS estaban Up.
2. **Lo esperado, por escrito.** Escribí qué esperaba antes de cada prueba y lo comparé con lo observado. Las diferencias están en cada documento.
3. **De lo físico a lo lógico.** Enlace, dirección y ruta, gateway, internet por IP, DNS y web, desde `cliente1`; y desde el lado del NOC, LibreNMS, `ping` desde `librenms`, la gestión y VirtualBox.
4. **Misma hora en todos los equipos.** Antes de empezar comprobé el reloj del PC, de las VMs y de GLPI (`America/Caracas`), para que las líneas de tiempo cuadren.
5. **Cierre completo.** Ticket con solución propuesta por `tecnico1`, aceptada por `usuario1` y cerrado, más un artículo nuevo en la base de conocimiento.

## El mismo síntoma, tres mensajes distintos

Para el usuario los tres incidentes son "no abre páginas". Desde `cliente1` se ven distintos:

| Incidente | Gateway (`10.10.10.1`) | Internet por IP | Mensaje del DNS | Qué indica |
|---|---|---|---|---|
| 01: CHR apagado | no responde | no | `No route to host` | El cliente no llega a su servidor DNS. |
| 02: relevo DNS desactivado | responde | sí | `Connection refused` | El servidor está alcanzable pero no atiende. |
| 03: pfSense apagado | responde | no | `SERVFAIL` | El relevo del CHR contesta, pero no obtiene respuesta de arriba. |

## Lo que dejaron los incidentes

- Un enlace en UP no prueba que el equipo de enfrente esté vivo (incidente 01, en VirtualBox).
- Un monitoreo que solo hace ping no ve un DNS roto (incidente 02).
- Detectar y avisar son caminos distintos: LibreNMS vio la caída de pfSense, pero el aviso a Telegram pasa por pfSense (incidente 03).
- Un error de transporte puede imprimir credenciales: la URL del error de Telegram incluía el token del bot. Lo regeneré y en los documentos y las capturas el token y el `chat_id` están tapados.

## Qué no se hizo ni se verificó

- No hay un canal de avisos que no dependa de pfSense, y LibreNMS no vigila DNS. Son límites conocidos, no resueltos.
- Causas no determinadas: por qué el ARP de `cliente1` mostró `DELAY` en el incidente 01 (predije `FAILED`); el origen de una entrada de caché de `debian.org`; qué DNS usa `librenms` (se supone pfSense, por DHCP); si LibreNMS reintenta un aviso fallido; y por qué `grep telegram` en `librenms.log` no devolvió nada.
- No anoté el desglose de los 27 min entre el aviso y el ticket n.º 3 (en un NOC real el ticket se abre al recibir la alerta).
- Cada documento tiene su propia sección "Qué no se verificó".

## Capturas

Archivos `.png` en [capturas/](capturas/), numerados del 01 al 34; las capturas largas están partidas en `a` y `b`.

| Incidente | Capturas |
|---|---|
| 01 | 01 a 17 (más 33 y 34 compartidas) |
| 02 | 18 a 25 (más 34) |
| 03 | 26 a 32 (más 34) |
| Comunes | 33 (hora y RAM después) y 34 (historial de alertas de LibreNMS) |
