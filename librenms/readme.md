# Bitácora de laboratorios

Notas propias de mis laboratorios, para repasar antes de entrevistas. Todo es de laboratorio, no de producción. Cuando algo no lo pude explicar o confirmar, lo marco.

> **Nota de actualización (20/11/2026).** Este laboratorio describe la red como estaba el día que lo hice. Después rediseñé la red del proyecto Network Operations Lab y algunos datos de abajo ya no son los actuales: la VM ya no va en puente (hoy es `192.168.10.13` en `lab-core` y `192.168.56.13` de gestión; la `192.168.1.102` ya no existe); la RAM bajó de 4 GB a 2 GB; y el dispositivo viejo sigue registrado en LibreNMS hasta el chat 5. El texto original no se modificó, para conservar lo que hice en su momento. Estado actual: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md).
---

## Laboratorio 1: Monitoreo con LibreNMS (nov. 2026)

### Resumen en 30 segundos
Armé en VirtualBox una máquina virtual con Ubuntu Server 24.04 e instalé LibreNMS (herramienta de monitoreo de red basada en SNMP). Seguí una guía paso a paso, con ayuda de IA, y diagnostiqué los errores que fueron saliendo. Agregué la propia VM y mi router de casa como dispositivos, y provoqué caídas de servicios para ver cómo se comporta el monitoreo. No es experiencia en producción.

### Entorno
- VirtualBox 7.2.22; antes tenía la 6.1, la desinstalé y empecé desde cero.
- VM: Ubuntu Server 24.04.5 LTS, 4 GB de RAM, 2 CPU, disco de 30 GB, red en modo puente (la VM tiene IP propia en mi red local).
- LibreNMS (canal monthly) con Nginx (servidor web), PHP 8.4 con PHP-FPM, MariaDB (base de datos) y SNMP.
- Acceso desde Windows por SSH y por el navegador.

### Cómo funciona (con mis palabras)
1. El navegador pide la página a Nginx.
2. Nginx le pasa el trabajo a PHP-FPM por un socket (un archivo que sirve de "puente").
3. PHP arma la pantalla y consulta a MariaDB los datos.
4. Los datos llegan a MariaDB porque el poller (proceso de LibreNMS que corre cada pocos minutos) le pregunta por SNMP a cada equipo. SNMP es el "idioma" con que el equipo responde; no es quien recolecta.
5. Las gráficas se guardan en archivos RRD (formato de series de tiempo), no en MariaDB.

### Problemas que resolví
| Problema | Causa | Solución |
|---|---|---|
| El instalador pedía PHP 8.4.1 o superior | Ubuntu 24.04 trae PHP 8.3; la versión actual de LibreNMS pide 8.4 | Instalé PHP 8.4 desde un repositorio externo (PPA) y repetí las dependencias de Composer |
| Error 502 al abrir la página | El pool de PHP-FPM de LibreNMS no estaba cargado; Nginx no encontraba el socket | Moví el archivo del pool a la carpeta de PHP 8.4 y reinicié PHP-FPM |
| Falló el módulo de Python (command_runner) | Faltaba ese paquete | Lo instalé |
| Scheduler y cron sin configurar | Pasos que me faltaban | Copié los archivos de servicio y activé el timer |
| Advertencias de `server_name` y `base_url` | Estaban con valores genéricos | Los puse con la IP de la VM |

Quedó pendiente: por qué la VM tardó en cargar el pool nuevo es causa no determinada (en mi caso bastó reiniciar PHP-FPM).

### Ejercicios que hice para entenderlo
1. **Recorrido de servicios.** Revisé el estado de Nginx, PHP-FPM, MariaDB y SNMP, los puertos donde escucha cada uno (80, 3306 y 161/UDP) y el socket de PHP.
2. **Romper y arreglar.**
   - Apagué PHP-FPM: error 502. El log de Nginx decía que no podía conectar al socket. Lo encendí y volvió.
   - Apagué SNMP: LibreNMS marcó la VM como caída (aunque estaba encendida) por la regla de que perder SNMP afecta la disponibilidad. Cayó a las 08:25 y se recuperó a las 08:30.
   - Apagué MariaDB: la página cargaba, pero daba error de conexión a la base de datos al iniciar sesión (no un 502).
3. **Monitorear algo real.** Agregué mi router por ICMP (ping), sin SNMP. Lo apagué 24 minutos: LibreNMS lo marcó caído a las 9:35 y generó la alerta "Device Down! Due to no ICMP response"; volvió a Up a las 10:00.

### Qué aprendí
- 502 = Nginx vivo, PHP no. Error de base de datos = Nginx y PHP vivos, MariaDB no. Página que no carga del todo = revisar antes red, VM o Nginx. El orden es de lo físico a lo lógico.
- "Down" en el monitoreo no siempre significa que el equipo esté apagado: puede ser que no responda SNMP.
- ICMP dice si el equipo responde; SNMP dice cómo está por dentro (CPU, puertos, tráfico).
- Las alertas están sin canal de salida (transports en none): solo se ven dentro del panel.

### Lo que no sé o no hice
- No lo he usado con equipos de un ISP ni en producción.
- No he configurado alertas por correo o Telegram.
- Una apagada del router de 3 minutos no generó alerta; la de 24 minutos sí. Causa no determinada.
- No sé leer a fondo la gráfica de ping del router (qué representa cada color).
- Zabbix: pendiente.

### Si me preguntan "¿lo instalaste solo?"
Seguí una guía con ayuda de IA y entendí el recorrido de cada servicio; los errores los diagnostiqué leyendo los logs y probando. Para verificar lo que me indicaba la IA, comparaba con los mensajes de error y con la documentación oficial.