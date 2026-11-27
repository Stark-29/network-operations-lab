# LibreNMS en el Network Operations Lab

**Fecha:** 22–23/11/2026

Monitoreo con **LibreNMS** de un laboratorio en VirtualBox: una red pequeña protegida por pfSense, con un router MikroTik CHR detrás y GLPI para los tickets. Este documento recoge cómo limpié el LibreNMS del primer laboratorio, cómo activé SNMP v2c en pfSense, el CHR y la propia VM, los dispositivos que registré por sus IPs de `lab-core`, las gráficas, las alertas por Telegram y tres pruebas de caída del CHR para medir cuánto tarda en verse una caída.

Diseño completo de la red: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md). Capturas en [`capturas/`](capturas/). La instalación de LibreNMS y los primeros ejercicios, antes de rediseñar la red, están en [`primer-laboratorio.md`](primer-laboratorio.md).

> **Nota de transparencia:** es un laboratorio en VirtualBox, seguido con ayuda de una IA (asistente de chat). No es experiencia en producción y no he operado LibreNMS en una red real ni con equipos de un ISP. Lo que no pude verificar está marcado como "no verificado" o "no probado", y las causas que no determiné quedan como "causa no determinada".
> 
> Nota de actualización (25/11/2026). En la etapa de incidentes apagué el CHR y pfSense con LibreNMS encendido (incidente-01 e incidente-03). Con el CHR apagado, LibreNMS avisó por Telegram. Con pfSense apagado, LibreNMS marcó el Down pero el aviso no salió: la resolución DNS de api.telegram.org agotó los 10 s. Ver la sección 6 y "Qué no se hizo".

## Entorno

| Elemento | Valor |
|---|---|
| Hipervisor | VirtualBox |
| VM `librenms` | Ubuntu Server 24.04, 2 GB de RAM; `lab-core` `192.168.10.13` y gestión `192.168.56.13` |
| LibreNMS | 26.12.0-dev.85 (según `validate.php`) con Nginx, PHP 8.4.27, MariaDB 10.11.14 y RRDTool 1.7.2 |
| SNMP en la VM | net-snmp 5.9.4.pre2 |
| Equipos monitoreados | `pfsense` (`.1`), `chr` (`.12`), `librenms` (`.13`) y `glpi` (`.14`), todos por su IP de `lab-core` |
| Canal de alertas | Bot de Telegram |

```
                 Internet ◄── api.telegram.org (HTTPS, alertas)
                    │
             pfsense  .1  ◄── SNMP UDP 161, solo desde .13
                    │
 lab-core 192.168.10.0/24 ─┬──────────────┬─────────────────┐
                           │              │                 │
                      librenms .13     glpi .14         chr .12 ◄── SNMP
                      (monitoreo,      (solo ping)
                       SNMP a sí mismo)
```

Durante todas las pruebas el PC anfitrión tuvo una VPN (NordVPN) encendida, como en el resto del proyecto. Las horas de las pruebas vienen del reloj de mi PC y del registro de eventos de LibreNMS; todos los equipos están en `America/Caracas`.

## Resumen

| Tema | Resultado | Estado |
|---|---|---|
| Dirección web | `server_name`, `base_url` y `APP_URL` con `192.168.56.13`; borré el dispositivo viejo (`192.168.1.102`) y el router de casa | Verificado (el resultado de `validate.php` está en la sección 1) |
| SNMP v2c | `pfsense`, `chr` y `librenms` responden a `snmpget` desde `librenms` | Verificado |
| Rechazo de SNMP desde otra IP | Probado el 24/11/2026 desde `cliente1`: sin respuesta hacia pfSense, el CHR y `librenms`, con control desde `librenms` ([pruebas](../integracion/pruebas.md)) | Verificado en pfSense (log); en el CHR y `librenms` la causa se deduce de la configuración |
| Dispositivos | 4 por IPs de `lab-core`: 3 con SNMP y `glpi` solo por ping | Verificado |
| Gráficas | CPU, memoria y tráfico de `pfsense`; tráfico por interfaz del `chr` | Verificado |
| Alertas | Telegram con una Operación asignada a la regla de caída por ICMP; incluye el aviso de recuperación | Verificado |
| Tiempo de detección | Chequeo cada 5 minutos; aviso entre 49 y 55 s después de detectar el cambio | Verificado en 7 transiciones (4 el 23/11 y 3 el 25/11); la causa de esos ~50 s, no verificada |
| Caída corta sin chequeo dentro | No deja Down ni alerta; solo un aviso de reinicio | Verificado (una vez) |
| Caída de pfSense | LibreNMS la marca Down (19:40:14), pero el aviso a Telegram no sale: falla la resolución DNS de api.telegram.org; al volver pfSense solo llega la recuperación | Verificado (una vez, 25/11/2026) |

## 1. Limpieza y dirección web

El LibreNMS del primer laboratorio quedó registrado con la IP de casa. Antes de cambiar nada, Nginx tenía `server_name 192.168.1.102;`, `base_url` valía `http://192.168.1.102/` y el archivo `.env` no tenía ninguna línea `APP_URL`. Dejé las tres con la dirección por la que entro desde mi PC, la de la red de gestión: `server_name 192.168.56.13 192.168.10.13;` en Nginx, `lnms config:set base_url http://192.168.56.13` y `APP_URL=http://192.168.56.13` en `.env`. Después `nginx -t` salió bien, recargué Nginx y reinicié `php8.4-fpm`.

`base_url` y `APP_URL` tienen que coincidir, y `APP_URL` es lo que LibreNMS usa para armar los enlaces de las gráficas en las alertas ([documentación](https://docs.librenms.org/Support/Environment-Variables/)).

Luego borré los dos dispositivos que había: `192.168.1.102` (la IP vieja de la VM, caída) y el router de casa, que monitoreaba por ping. Desde `lab-core` la regla `lab no ve la casa` impide llegar a esa red, así que habría quedado caído para siempre ([captura de antes](capturas/04-dispositivos-antes.png)).

**Validación.** Corrí `./validate.php` el 22/11/2026, justo después del cambio, y no salió limpio: tres `FAIL` (`The poller is not running`, `No active python wrapper pollers found` y `Scheduler is not running`) y un `WARN` (la instalación llevaba más de 24 horas sin actualizarse). No investigué los `FAIL` en ese momento. Más tarde, con las VMs encendidas desde hacía horas, LibreNMS ya hacía sus comprobaciones cada 5 minutos (se ven en el registro de eventos, sección 6), así que el poller y el planificador sí corrían. Repetí `validate.php` el 23/11/2026: sin `FAIL;` solo queda el `WARN` de actualización.

## 2. SNMP v2c en los tres equipos

`librenms` consulta por SNMP a `pfsense`, a `chr` y a sí mismo. La versión v2c manda la "comunidad" (una especie de contraseña) sin cifrar, así que hice tres cosas: usé una comunidad propia y no `public`, usé la misma para los tres (ver "Qué no se hizo") y en cada equipo la limité a la IP de `librenms` (`192.168.10.13`). La comunidad no está en este repositorio: donde aparece, la escribo `<COMUNIDAD>`.

**pfSense** (Services → SNMP): servicio habilitado, puerto 161, ubicación y contacto, comunidad propia, módulos MibII, PF y Host Resources (este último necesita MibII) y el servicio atado solo a la interfaz LAN ([captura](capturas/01-pfsense-snmp.png), con la comunidad tapada). `sockstat -4 -l -p 161` muestra a `bsnmpd` escuchando en `192.168.10.1:161` y no en todas las direcciones ([documentación de Netgate](https://docs.netgate.com/pfsense/en/latest/services/snmp.html)).

La regla de bloqueo `lan no administra pfsense` ya impedía todo lo que la LAN dirigía a pfSense salvo ping, DNS y NTP, así que añadí una regla en la pestaña LAN: Pass · UDP · origen `192.168.10.13` · destino LAN address · puerto 161 · `snmp desde librenms` ([captura](capturas/03-pfsense-regla-snmp.png), con contadores). Tiene que ir por encima del bloqueo, y el SNMP funcionó, así que lo está. No probé a quitarla para ver el bloqueo.

**CHR** (RouterOS 7.25.1):

```
/snmp set enabled=yes contact="Robert" location="Laboratorio VirtualBox"
/snmp community set [find default=yes] name=<COMUNIDAD> addresses=192.168.10.13/32 security=none read-access=yes write-access=no
```

RouterOS trae una comunidad `public` por defecto ([documentación de MikroTik](https://help.mikrotik.com/docs/spaces/ROS/pages/8978519/SNMP)); la renombré a la mía y la limité a una dirección en vez de dejarla. Comprobé que ya no existe ninguna llamada `public` (`:put [:len [/snmp community find name=public]]` dio `0`) y que las direcciones permitidas son solo `192.168.10.13/32`. Ojo: `/snmp print` muestra el nombre de la comunidad en el campo `trap-community`, así que no publico esa salida sin redactar.

**`librenms`:** reemplacé `/etc/snmp/snmpd.conf` por uno mínimo (la copia anterior quedó en `snmpd.conf.bak-monitoreo`):

```
agentaddress udp:161
rocommunity <COMUNIDAD> 192.168.10.13
rocommunity <COMUNIDAD> 127.0.0.1
sysLocation Laboratorio VirtualBox
sysContact Robert
```

`ss -lunp` muestra a `snmpd` escuchando en `0.0.0.0:161`, es decir, en todas las IPv4 de la VM; quién puede preguntar lo deciden las líneas `rocommunity`.

**Prueba.** Desde `librenms` pregunté el nombre y la descripción del equipo (OIDs `1.3.6.1.2.1.1.5.0` y `1.3.6.1.2.1.1.1.0`) a los tres, y los tres respondieron ([captura](capturas/02-snmpget-tres-equipos.png)). La comunidad la guardé en una variable de shell con `read -rs`, que no muestra lo que escribo, para que no quede en el historial ni en las capturas.

## 3. Dispositivos en LibreNMS

| Dispositivo | IP | Método | Detectado |
|---|---|---|---|
| `pfsense` | 192.168.10.1 | SNMP v2c y ICMP | pfSense 2.9.1-RELEASE (amd64) |
| `chr` | 192.168.10.12 | SNMP v2c y ICMP | MikroTik RouterOS 7.25.1 |
| `librenms` | 192.168.10.13 | SNMP v2c y ICMP | Linux 6.8.0-149-generic |
| `glpi` | 192.168.10.14 | Solo ping | "Ping only" |

La comunidad se guardó una sola vez como secreto de LibreNMS y se reutilizó en los otros dos equipos con SNMP. `glpi` no tiene SNMP: lo registré con el SNMP apagado y le cambié el nombre a minúsculas para que coincida con los demás. Dejé activados "Validate on add" y "Affects availability"; según la documentación, este último hace que un ping fallido marque el equipo como caído ([documentación](https://docs.librenms.org/Support/Configuration)).

Los tres equipos con SNMP salieron con barra naranja en la lista porque llevaban menos de 24 horas encendidos; LibreNMS lo marca como advertencia ([captura](capturas/05-dispositivos-despues.png)). `glpi` salió en verde porque, al ser solo ping, LibreNMS no conoce su uptime.

## 4. Gráficas

Con Host Resources y MibII, pfSense entrega CPU, memoria y tráfico por interfaz ([captura](capturas/06-grafica-pfsense.png)). Para ver una gráfica con movimiento, desde `cliente1` corrí `ping -s 1400 -i 0.2 -w 1200 9.9.9.9`: 5956 enviados y 5956 recibidos, 0 % de pérdida, 40,6 ms de promedio. Eso son unos 5 paquetes de 1400 bytes por segundo, alrededor de 56 kbps por sentido. En `ether2` del CHR se ven unos 50 k por sentido ([captura](capturas/07-grafica-chr.png)) y la vista "Real time" (cada 5 segundos) marcó 54 y 56 Kbps; en el tráfico general de pfSense se ven unos 100 k porque suma interfaces. No guardé captura de la vista "Real time".

Las gráficas tienen un hueco de varias horas por un corte de luz en mi casa, con las VMs apagadas.

## 5. Alertas por Telegram

**Canal.** Elegí Telegram y no el correo porque el token de un bot se puede revocar en BotFather y no da acceso a ningún buzón; enviar por Gmail habría pedido una contraseña de aplicación de una cuenta de correo mía. El token y el ID del chat están solo en LibreNMS, no en el repo. El transporte `telegram-lab` (Alerts → Alert Transports) es de tipo Telegram, con formato HTML, el campo de hilo (Thread ID) en 0 (el formulario pedía un número entero; no verifiqué qué hace ese valor, y el mensaje llegó bien) y "Default alert" apagado. El botón de prueba mandó un mensaje al celular ([captura](capturas/09-telegram-prueba.png)). Antes comprobé desde la VM que el nombre `api.telegram.org` resuelve y que `curl` recibe una respuesta HTTP 302 (una redirección; lo que importaba era llegar y no recibir `000`) ([documentación](https://docs.librenms.org/Alerting/Transports/Telegram)).

**Operación.** En esta versión de LibreNMS una regla sin Operación levanta la alerta pero no notifica a nadie ([documentación](https://docs.librenms.org/Alerting/Operations/)). Mi lista de reglas decía "No operation (notifications suppressed)" en todas, y el desplegable "Operation" de la regla salía vacío porque no existía ninguna ([captura](capturas/08-reglas-device-down.png)). Creé la Operación `notificar-telegram`: un segmento, pasos 1 a 1, inicio en 0 s, duración de paso 60 s y el transporte `telegram-lab` ([captura](capturas/08c-operacion-telegram.png)). La "duración de paso por defecto" quedó en 0; supuse que solo se usa en segmentos sin duración propia (no verificado) y aquí no influye, porque el segmento tiene un solo paso.

La asigné **solo** a la regla `Device Down! Due to no ICMP response.`, cuya condición es `macros.device_down = 1 AND devices.status_reason LIKE '%icmp%'` ([captura de la consulta](capturas/08b-regla-icmp-avanzado.png)). En esta versión la regla no tiene una espera propia; los avisos de recuperación (`Recovery alerts`) están activados. La regla `Device Down (SNMP unreachable)` y las demás siguen sin Operación ([captura](capturas/08d-reglas-con-operacion.png)). El mensaje usa la plantilla por defecto; el campo `Alert sent to:` sale vacío y el mensaje llega igual.

## 6. Pruebas de caída del CHR

Apagué el CHR con `/system shutdown` por SSH y lo encendí desde VirtualBox. Las horas de apagado y encendido son de mi reloj; el resto, del registro de eventos de LibreNMS ([captura](capturas/10-eventlog-chr.png)).

| Prueba | Apagado | Encendido | Down detectado | Alerta (registro) | Up detectado | Recuperación |
|---|---|---|---|---|---|---|
| 1. Larga | 17:30:55 | 17:33:50 (se colgó; ver abajo) | 17:35:13 | 17:36:03 | 17:45:10 | 17:46:02 |
| 2. Corta, sin chequeo dentro | 17:55:40 | 17:58:30 (arrancó 17:58:45) | no | no | no | no |
| 3. Corta, con chequeo dentro | 18:04:30 | ~18:07:40 | 18:05:13 | 18:06:03 | 18:10:09 | 18:11:03 |

**Prueba 1.** Al encender el CHR se quedó en *load system* y no arrancó. Un reinicio desde VirtualBox lo recuperó en unos 20 s; el uptime registrado a las 17:45:10 fue de 243 s, o sea que arrancó hacia las 17:41. La causa del cuelgue no está determinada. La alerta llegó a Telegram a las 17:36:02 ([captura](capturas/11-telegram-alerta.png)) y el aviso de recuperación a las 17:46, con `Time elapsed: 10m 3s`.

**Prueba 2.** El CHR estuvo caído unos 3 min 5 s y LibreNMS no registró ningún Down ni mandó nada a Telegram. Lo único que quedó en el registro fue `Device rebooted after 14 minutes -> 82s` a las 18:00:07. Ese "14 minutos" es el uptime del último chequeo anterior, que encaja con uno hacia las 17:55:10, unos 30 s antes de apagarlo. El siguiente fue el de las 18:00 y el CHR ya había vuelto: la caída entera cayó entre dos chequeos.

**Prueba 3.** Caída de unos 3 min 10 s, casi igual que la 2, pero ahora el chequeo de las 18:05:13 cayó dentro: hubo Down, alerta y recuperación.

Lo que observé:

- Las detecciones (17:35:13, 17:45:10, 18:00:07, 18:05:13 y 18:10:09) caen entre el segundo 7 y el 13 de minutos terminados en 0 o 5. Es el chequeo cada 5 minutos: [por defecto, LibreNMS comprueba el ICMP con la frecuencia del poller](https://docs.librenms.org/Extensions/Fast-Ping-Check), normalmente 5 minutos. Existe un modo de "ping rápido" para chequear cada minuto; no lo activé.
- El aviso (alerta o recuperación) salió entre 50 y 54 s después de cada detección, en las cuatro transiciones. Eso encaja con un motor de alertas que corre una vez por minuto, y no con una espera de la regla, porque el inicio de la Operación es 0 s. No revisé la configuración del planificador, así que es consistente con lo observado pero no verificado.
- Desde que se apagó el CHR hasta que llegó la alerta pasaron 5 min 7 s en la prueba 1 y 1 min 33 s en la 3. El peor caso sería de unos 6 minutos (5 de chequeo y 1 del motor de alertas); es una estimación, no la medí.
- Lo que decide si hay alerta no es cuánto dura la caída, sino si un chequeo cae dentro de ella (pruebas 2 y 3).

**Lo que explica del primer laboratorio.** Allí una caída de 3 minutos del router no generó alerta y una de 24 minutos sí. Lo más probable es que fuera esto mismo, pero aquel laboratorio no anotó las horas de los chequeos, así que no reproduje el caso exacto: queda como explicación probable.

### Pruebas del 25/11/2026 (etapa de incidentes)

| Prueba | Apagado | Encendido | Down detectado | Alerta (registro) | Up detectado | Recuperación |
|---|---|---|---|---|---|---|
| 4. CHR, larga | 18:05:38 | 18:45:07 (arrancó ≈18:45:32) | 18:10:13 | 18:11:02 | 18:50:07 | 18:51:02 |
| 5. pfSense | 19:38:58 | 19:48:04 (arrancó ≈19:48:38) | 19:40:14 | no salió (error 19:41:14) | 19:50:08 | 19:51:03 |

- **Prueba 4.** El apagado cayó 25 s después de un chequeo (18:05:13) y se vio 4 min 35 s más tarde. El encendido cayó justo después del chequeo de las 18:45, con el CHR aún arrancando, y el Up se vio a las 18:50:07, unos 4 min 35 s después de que arrancara: el panel va con retraso en los dos sentidos. Del apagado a la alerta pasaron 5 min 24 s, cerca del peor caso estimado (unos 6 minutos). El CHR arrancó sin colgarse.
- **Prueba 5.** LibreNMS vio la caída 76 s después de apagar pfSense. El aviso falló con `cURL error 28: Resolving timed out after 10000 milliseconds` al contactar `api.telegram.org`, y no hubo reintento visible. Detalle: [incidente-03](../incidentes/incidente-03.md).
- **Avisos.** Los tres avisos que salieron el 25/11 tardaron 49, 55 y 55 s desde la detección. Con los cuatro del 23/11 son 7 de 7 entre 49 y 55 s, todos registrados en el segundo 02 o 03 de un minuto (17:36:03, 17:46:02, 18:06:03, 18:11:03, 18:11:02, 18:51:02 y 19:51:03). Sigue siendo consistente con un motor de alertas que corre una vez por minuto, sin verificar.

## Problemas y rarezas

1. **Dirección web con la IP vieja:** `server_name` y `base_url` apuntaban a `192.168.1.102` y `.env` no tenía `APP_URL` (sección 1).
2. **Timeout de SNMP solo en `librenms`:** `pfsense` y `chr` respondían, pero `192.168.10.13` daba `Timeout: No Response`. El `snmpd.conf` se había generado con una variable `COM` equivocada: el `read` se quedó esperando entrada y guardó otro texto. Volver a escribir la comunidad y regenerar el archivo lo arregló. Es la causa más probable; no abrí el archivo para comprobarla. SNMP no avisa de una comunidad incorrecta: simplemente no responde.
3. **Desplegable "Operation" vacío:** faltaba crear la Operación (sección 5).
4. **El CHR se colgó al arrancar** tras la primera prueba: causa no determinada (sección 6).
5. **`/snmp print` muestra la comunidad** en `trap-community`: la salida se redacta antes de publicarla.
6. **`validate.php` con tres `FAIL`** el 22/11/2026 (sección 1).
7. **El registro de eventos mostró el token del bot** dentro del mensaje de error de Telegram (sección 6): lo regeneré con /revoke y tapé las capturas.

## Lo que aprendí

- Una alerta no es una notificación: en esta versión, sin una Operación asignada, la regla levanta la alerta y nadie se entera.
- Un monitoreo solo ve una caída si un chequeo cae dentro de ella. El intervalo de chequeo fija cuánto tarda en verse una caída y qué caídas cortas pasan sin dejar rastro.
- Si el equipo responde al ping pero `snmpget` da *Timeout*, el problema no es la ruta: es la comunidad, el servicio o una regla que bloquea el puerto.
- Antes de pegar la salida de un comando en una captura o en un repo, hay que saber si imprime secretos (aquí, `/snmp print`).
- Un comando que se queda esperando entrada (`read`) se ejecuta solo y se comprueba antes de seguir; lo que se pegue después puede acabar dentro de la variable.
- Detectar la caída es una cosa y avisar es otra: entre el Down de LibreNMS y el mensaje en el celular pasó casi un minuto más.
- Detectar y avisar son caminos distintos: LibreNMS vio caer a pfSense, pero el aviso depende de pfSense y no llegó. Un aviso que falla no hace ruido.
- Un mensaje de error de transporte puede llevar credenciales: la URL del error de Telegram incluía el token del bot. Se revisa antes de copiarla o capturarla.

## Qué no se hizo

- SNMP v3 (con autenticación y cifrado): solo v2c, con la comunidad sin cifrar.
- Una sola comunidad para los tres equipos.
- Confirmar con un log del equipo el rechazo de SNMP desde una IP no autorizada en el CHR y en librenms: el 24/11/2026 no hubo respuesta en los tres equipos, pero la causa se deduce de la configuración ([pruebas](../integracion/pruebas.md)).
- Solo la regla de caída por ICMP notifica; `Device Down (SNMP unreachable)`, `Device rebooted`, puertos y latencia no.
- Activar el ping rápido (chequeo cada minuto): el chequeo es cada 5 minutos.
- Un camino de avisos que no dependa de pfSense: el 25/11/2026 comprobé que, con pfSense apagado, LibreNMS marca el Down pero el aviso a Telegram no sale ([incidente-03](../incidentes/incidente-03.md)).
- SNMP en `glpi`: solo se monitorea por ping.
- Actualizar LibreNMS: el aviso de "más de 24 horas sin actualizar" quedó sin atender.
- Sin experiencia en producción con LibreNMS ni con equipos de un ISP.

## Si me preguntan "¿lo configuraste solo?"

Seguí la documentación de LibreNMS, pfSense y MikroTik con ayuda de una IA. Cuando algo no respondía separé el problema por capas: si el equipo contesta al ping, si el servicio escucha, si el firewall deja pasar el puerto y si la comunidad coincide. Para no creerle todo a la IA, comprobé cada afirmación con una prueba (`snmpget`, `sockstat`, apagar el CHR y mirar el registro de eventos) o con la documentación oficial.

## Referencias

- LibreNMS, [Alerting](https://docs.librenms.org/Alerting/) y [Operations](https://docs.librenms.org/Alerting/Operations/).
- LibreNMS, [Telegram transport](https://docs.librenms.org/Alerting/Transports/Telegram).
- LibreNMS, [Fast Ping Check](https://docs.librenms.org/Extensions/Fast-Ping-Check).
- LibreNMS, [Configuration](https://docs.librenms.org/Support/Configuration).
- LibreNMS, [Environment Variables](https://docs.librenms.org/Support/Environment-Variables/).
- Netgate, [SNMP](https://docs.netgate.com/pfsense/en/latest/services/snmp.html).
- MikroTik, [SNMP](https://help.mikrotik.com/docs/spaces/ROS/pages/8978519/SNMP).

## Versiones y herramientas

- LibreNMS 26.12.0-dev.85, PHP 8.4.27, MariaDB 10.11.14, RRDTool 1.7.2, net-snmp 5.9.4.pre2, Python 3.12.3, Composer 2.11.2
- pfSense CE 2.9.1-RELEASE (amd64) y MikroTik RouterOS 7.25.1 (`chr`)
- Ubuntu Server 24.04 (`librenms`, `glpi`, `cliente1`)
- VirtualBox, bot de Telegram y PowerShell con OpenSSH
