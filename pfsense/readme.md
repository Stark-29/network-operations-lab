# pfSense en el Network Operations Lab

**Fecha:** 21/11/2026

Firewall/router **pfSense CE** de un laboratorio en VirtualBox: es el borde de una red pequeña (`lab-core`), con un router MikroTik CHR detrás, monitoreo con LibreNMS y tickets en GLPI. Este documento recoge la configuración de pfSense y las pruebas que le hice: SSH por una interfaz de gestión, NAT saliente, DNS, reglas de firewall de la LAN, respaldo con restauración probada, reinicio y NTP interno. Las pruebas de red se hicieron desde `librenms`, `glpi` y `cliente1`.

Diseño completo de la red: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md). Capturas en [`capturas/`](capturas/). La primera toma de contacto con pfSense, antes de rediseñar la red, está en [`primer-laboratorio.md`](primer-laboratorio.md).

> **Nota de transparencia:** es un laboratorio en VirtualBox, seguido con ayuda de una IA (asistente de chat). No es experiencia en producción y no he operado pfSense en una red real. Lo que no pude verificar está marcado como "no verificado" o "no probado", y las causas que no determiné quedan como "causa no determinada".
>
> **Nota de actualización (22/11/2026).** Este documento describe pfSense como quedó el 21/11/2026. En la etapa de integración cambió: hay un gateway CHR_LAB (192.168.10.12) y dos rutas estáticas hacia 10.10.10.0/24 y 10.10.20.0/24; el CHR dejó de enmascarar, así que el NAT lo hace solo pfSense (ya no hay doble NAT); y las reglas de la LAN usan el alias LAB_NETS como origen, con una regla nueva "lab a internet". Detalle y pruebas: [integración](../integracion/readme.md). Las capturas y el config redactado de esta carpeta son del 21/11/2026. El 23/11/2026 se añadió el servicio SNMP (solo en la LAN) y la regla `snmp desde librenms` (UDP 161, origen `192.168.10.13`); detalle en [librenms](../librenms/readme.md).

## Entorno

| Elemento | Valor |
|---|---|
| Hipervisor | VirtualBox |
| pfSense | CE 2.9.1-RELEASE (amd64) |
| Recursos | 1 GB de RAM; CPU y disco como en el [primer laboratorio](primer-laboratorio.md#entorno) |
| `em0` (WAN) | NAT de VirtualBox, `10.0.2.15/24` por DHCP |
| `em1` (LAN) | Red interna `lab-core`, `192.168.10.1/24`; DHCP en `.100`-`.150` |
| `em2` (GESTION) | Solo-anfitrión, `192.168.56.11/24`, sin gateway |

```
Internet ── [NAT de VirtualBox] ── em0 WAN 10.0.2.15
                                      │
   PC anfitrión ── em2 GESTION ── [ pfSense ]
   (192.168.56.1)  192.168.56.11      │
                                  em1 LAN 192.168.10.1
                                      │
        lab-core 192.168.10.0/24 ─────┼──────────────┐
                                      │              │
                         librenms .13 · glpi .14   chr .12 ── lab-clientes ── cliente1
```

Durante todas las pruebas el PC anfitrión tuvo una VPN (NordVPN) encendida, salvo donde se indica. Resultó afectar al DNS (sección 4).


## Resumen

| Tema | Resultado | Estado |
|---|---|---|
| SSH | Habilitado; puerto 22 abierto solo desde `192.168.56.0/24` (interfaz GESTION) | Verificado |
| NAT saliente | Modo automático; traducción observada en la tabla de estados | Verificado |
| DNS | Único servidor `10.0.2.3` y reenvío activado; con la VPN encendida fallan `1.1.1.1` y `8.8.8.8` | Verificado; mecanismo de la VPN sin determinar |
| Reglas de la LAN | 6 reglas propias: ping, DNS y NTP al gateway, SNMP desde librenms (23/11/2026), bloqueo a pfSense y bloqueo a la red de casa | Verificado; el puerto 22 con el antilockout activo no se probó |
| Antilockout | Desactivado; la administración queda solo por GESTION | Verificado |
| Respaldo | Restauración probada con un cambio real | Verificado |
| Reinicio | Unos 1 min 40 s; todo volvió igual | Verificado |
| Reservas DHCP | 3 reservas documentadas | Verificado |
| NTP interno | pfSense da la hora a `glpi`, `librenms` y `chr` | Verificado |

## 1. Respaldo inicial

Antes de tocar nada: snapshot de VirtualBox `pfsense-inicio` (VM apagada) y copia de `config.xml` (Diagnostics → Backup & Restore, Backup area: All). La copia contiene contraseñas, hashes y llaves, así que queda **fuera del repositorio**. Al terminar, snapshot `reglas-listo`.

## 2. SSH por la interfaz GESTION

pfSense se administraba por web desde GESTION (solo TCP 443). Para sumar SSH:

1. Regla en la pestaña GESTION, copia de la de 443 con el puerto cambiado: Pass · TCP · origen `192.168.56.0/24` · destino GESTION address · puerto 22 · `ssh desde gestion` ([captura](capturas/02-regla-gestion-ssh.png)).
2. System → Advanced → Admin Access → Enable Secure Shell, autenticación por contraseña ([captura](capturas/01-ssh-habilitado.png)). El log mostró que pfSense generó las llaves SSH que faltaban.
3. `ssh admin@192.168.56.11` desde PowerShell conectó a la primera. Comparé la huella ED25519 que mostró el cliente con la que da `ssh-keygen -lf /etc/ssh/ssh_host_ed25519_key.pub` en pfSense, y coincidieron.
4. `sockstat -4 -l -p 22` muestra `sshd` escuchando en `*:22`, es decir, en todas las direcciones IPv4 ([captura](capturas/03-sshd-escucha.png)). El servicio no se limita por interfaz: quién llega lo deciden las reglas.

## 3. NAT saliente automático

El modo de Outbound NAT está en **automático**: pfSense genera una regla por cada subred interna que traduce el origen a la IP de la WAN (`10.0.2.15`). `pfctl -s nat` mostró reglas para `127.0.0.0/8`, `192.168.10.0/24` y `192.168.56.0/24`, cada una con puertos `1024:65535`, y una copia de cada una para el puerto 500 (ISAKMP) con *static port*, pensada para que IPsec pueda pasar (este laboratorio no usa IPsec) ([captura](capturas/04-nat-saliente-automatico.png)).

Demostración con la tabla de estados:

- Un `ping` de `librenms` a `9.9.9.9` aparece como `10.0.2.15:28790 (192.168.10.13:17352) -> 9.9.9.9`: la IP traducida va fuera del paréntesis y la original dentro ([captura](capturas/05-nat-estado-librenms.png)).
- El mismo `ping` desde `cliente1` a `8.8.4.4` aparece con origen `192.168.10.12`, que es el CHR, no `cliente1` ([captura](capturas/06-nat-estado-cliente1.png)). El CHR enmascara antes, así que al 21/11/2026 hay **doble NAT**.
- Los 60 pings de una prueba generan un solo estado, no 60: pf registra flujos, no paquetes.

## 4. DNS

**Origen de los DNS `103.86.x.x` del dashboard** (quedaron sin explicar en el primer laboratorio). El lease de la WAN (`/var/db/dhclient.leases.em0`) trae `103.86.96.100, 103.86.99.100, 1.1.1.1, 8.8.8.8` ([captura](capturas/07-dns-lease-wan.png)). El DHCP del NAT de VirtualBox ofrece al invitado los DNS que tiene configurados el anfitrión, y `ipconfig /all` en mi PC muestra esos dos `103.86` en el adaptador NordLynx de NordVPN. Cadena: VPN del PC → DNS del anfitrión → DHCP de VirtualBox → WAN de pfSense → dashboard.

**Efecto de la VPN.** Con "Allow DNS server list to be overridden by DHCP/PPP on WAN" desmarcado y el DNS Resolver en modo reenvío hacia `1.1.1.1` y `8.8.8.8`, `librenms` resolvía o fallaba según el estado de la VPN:

| Override | VPN | Resolución desde `librenms` |
|---|---|---|
| desmarcado | apagada | funciona |
| marcado | encendida | funciona |
| desmarcado | encendida | `SERVFAIL` |

Consultando directamente desde el shell de pfSense con `drill`:

| Servidor | VPN encendida | VPN apagada |
|---|---|---|
| `1.1.1.1` (UDP) | falla | responde |
| `8.8.8.8` (UDP) | falla | no probado |
| `1.1.1.1` (TCP) | falla | no probado |
| `10.0.2.3` (DNS interno del NAT de VirtualBox) | responde | responde |
| `103.86.96.100` (DNS de NordVPN) | responde | no probado |

([captura 10b](capturas/10b-dns-drill-contraste.png)). Lectura: con la VPN encendida, los DNS públicos no responden desde el laboratorio, pero `10.0.2.3` y los DNS de NordVPN sí. Con el override marcado funcionaba, probablemente porque Unbound también reenviaba a los DNS del lease que sí contestan (explicación probable, no verificada). **No determiné cómo la VPN causa esto.**

**Configuración final:** en System → General Setup, `10.0.2.3` como único servidor DNS, sin gateway; override desmarcado; reenvío activado ([captura](capturas/08-general-setup-dns.png), [captura](capturas/09-dns-resolver-reenvio.png)). Verificada con la VPN encendida ([captura](capturas/10-dns-prueba-final.png)). Dependo del DNS interno de VirtualBox; no verifiqué cómo resuelve por dentro.

**Por qué hizo falta el modo reenvío en el primer laboratorio: causa no determinada, no reproducida.** Con el reenvío desactivado, Diagnostics → DNS Lookup resolvió `archlinux.org` (VPN apagada) y `gnu.org` (VPN encendida). DNSSEC estaba activado y no lo toqué. Esto contradice mi sospecha original (que la resolución desde los servidores raíz fallaba detrás del NAT), así que esa sospecha queda sin respaldo. Después reactivé el reenvío.

## 5. Reglas de firewall de la LAN

pfSense evalúa las reglas de arriba hacia abajo y gana la primera que coincide; lo que ninguna permite, se bloquea. Además existe una regla oculta de **antilockout** que pfSense coloca en la LAN y que deja entrar a la web y al SSH desde la LAN pase lo que pase en las demás reglas. Como GESTION es un canal de administración aparte, la desactivé al final (System → Advanced → Admin Access) y dejé que las reglas decidan quién administra pfSense.

Reglas añadidas en la pestaña LAN el 21/11/2026, todas IPv4 y con origen LAN subnets (desde el 22/11 usan el alias LAB_NETS; ver la nota de actualización) ([captura](capturas/17b-reglas-lan-final.png), con contadores):

| Acción | Protocolo | Destino | Log | Descripción | Para qué |
|---|---|---|---|---|---|
| Pass | ICMP (echo request) | LAN address | no | `ping al gateway` | Primer paso del diagnóstico "usuario sin internet"; LibreNMS lo usará para monitorear |
| Pass | TCP/UDP 53 | LAN address | no | `dns al gateway` | Los equipos usan a pfSense como DNS |
| Pass | UDP 123 | LAN address | no | `ntp al gateway` | NTP interno (sección 9); va por encima del bloqueo |
| Pass | UDP 161 | LAN address | no | `snmp desde librenms` | Solo desde `192.168.10.13`: LibreNMS consulta el SNMP de pfSense |
| Block | Any | This Firewall (self) | sí | `lan no administra pfsense` | La LAN no llega a la web ni al SSH de pfSense, ni por su IP LAN, WAN o GESTION |
| Block | Any | Network `192.168.1.0/24` | sí | `lab no ve la casa` | El NAT de VirtualBox deja llegar a la red de casa; el laboratorio no debe tocarla |

Debajo quedan las reglas de fábrica ("Default allow LAN to any" y su par IPv6), sin tocar: dan salida libre a internet. Usé **Block** y no Reject: descarta en silencio y el cliente ve un *timeout*. Para una subred el tipo de destino debe ser **Network**, no "Address or Alias".

Pruebas desde `librenms` (VPN encendida; "—" = no repetido) ([base](capturas/11-prueba-base-antes.png), [con reglas](capturas/12-prueba-reglas-despues.png), [sin antilockout](capturas/15-prueba-sin-antilockout.png)):

| Prueba | Sin reglas | Con reglas, antilockout activo | Antilockout desactivado |
|---|---|---|---|
| `ping 192.168.10.1` | responde | responde | — |
| `resolvectl query wikipedia.org` | resuelve | resuelve | — |
| `nc -zv 192.168.10.1 443` | abre | abre | *timeout* |
| `nc -zv 192.168.10.1 22` | abre | **no probado** | *timeout* |
| `nc -zv 10.0.2.15 443` (WAN de pfSense) | abre | *timeout* | — |
| `ping 192.168.1.1` (router de casa) | responde | 100 % pérdida | — |

- Con el antilockout activo, el 443 de `192.168.10.1` sigue abriendo aunque el bloqueo exista: lo permite la regla oculta. Al desactivarla, el 443 y el 22 pasan a *timeout*, y la administración desde GESTION siguió funcionando ([captura](capturas/14-antilockout-desactivado.png)).
- **Control:** con las reglas de ping y DNS deshabilitadas, el ping al gateway da 100 % de pérdida y el DNS da *Connection timed out*; las atrapa el bloqueo ([captura](capturas/16-control-r1-r2-deshabilitadas.png)). Las volví a habilitar.
- DHCP: tras `sudo networkctl renew enp0s3`, `glpi` conservó `192.168.10.14` ([captura](capturas/13-dhcp-renovacion-glpi.png)). No comprobé que la renovación terminara. Creo que pfSense permite su propio DHCP aparte de las reglas del usuario, pero no lo verifiqué.
- Los bloqueos quedaron en el log ([captura](capturas/18-log-bloqueos.png)).

**Nota:** las pruebas de este documento se hicieron con las reglas en LAN subnets. El 22/11/2026 se cambió el origen a LAB_NETS (lab-core, lab-clientes y la VLAN 10) para cubrir a los clientes detrás del CHR, que ahora llegan con su IP real; ver [integración](../integracion/readme.md)

## 6. Respaldo, restauración y copia sin secretos

1. Respaldo de `config.xml` (sin cifrar, fuera del repo).
2. Cambio real: deshabilité `lab no ve la casa`; el ping a la casa volvió a responder ([captura](capturas/19-restauracion-antes.png)).
3. Restauré el respaldo desde Diagnostics → Backup & Restore ([captura](capturas/20-restauracion-despues.png)). La regla volvió a estar habilitada y el ping volvió a bloquearse ([captura](capturas/21-restauracion-pings.png)). Es una restauración probada por comportamiento, no solo "el archivo se subió".

**Copia para el repositorio:** [`configs/config-pfsense-redactado.xml`](configs/config-pfsense-redactado.xml), generada del respaldo final (21/11/2026, con la regla de NTP). Se reemplazó por `<REDACTED>` el **valor** de 11 elementos: las 4 llaves SSH del servidor (`xmldata`), la llave privada y el certificado de la web (`prv`, `crt`), el hash de la contraseña de admin, la comunidad SNMP y 3 nombres de algoritmo DDNS (redacción de más, inofensiva). No se borró nada más: reglas, interfaces, IPs, MAC de las reservas y descripciones siguen visibles. Esa copia **no sirve para restaurar** pfSense. Los respaldos originales quedan fuera del repo. Se generó con un script de PowerShell propio (no incluido) y dos revisiones, una por nombre de etiqueta y otra por contenido; las dos terminaron sin resultados.

## 7. Reinicio

Reinicio desde Diagnostics → Reboot: unos 1 min 40 s hasta que volvió. Comprobé que los servicios `unbound`, `dhcpd` y `sshd` estaban en ejecución ([captura](capturas/23-servicios-tras-reinicio.png)), que las tres interfaces estaban arriba y las reglas intactas ([captura](capturas/22-dashboard-tras-reinicio.png)), que `chr`, `librenms` y `glpi` conservaron sus IPs y que `cliente1` seguía saliendo a internet por el CHR y pfSense.

## 8. Reservas DHCP

Las reservas se hicieron al diseñar la red; aquí las documento. Están fuera del pool `192.168.10.100`–`192.168.10.150` ([reservas](capturas/24-reservas-dhcp.png), [rango](capturas/25-dhcp-rango-lan.png)). Las MAC están en [direccionamiento](../docs/direccionamiento.md#4-reservas-en-pfsense).

| Equipo | IP reservada |
|---|---|
| `chr` | `192.168.10.12` |
| `librenms` | `192.168.10.13` |
| `glpi` | `192.168.10.14` |

La IP va en el campo **IP Address** de la reserva, no en *Client identifier*. Si se regenera la MAC de una VM, hay que actualizar su reserva.

## 9. NTP interno

`ntpq -p` en pfSense mostró varias fuentes externas con *reach* 377 y una elegida (marcada con `*`). Con la regla de UDP 123 en la LAN, los clientes usan a pfSense como servidor de hora:

- `glpi` y `librenms`: `/etc/systemd/timesyncd.conf.d/lab.conf` con `NTP=192.168.10.1` ([glpi](capturas/26-ntp-glpi.png), [librenms](capturas/27-ntp-librenms.png)).
- `chr`: `/system ntp client set enabled=yes servers=192.168.10.1` ([captura](capturas/28-ntp-chr.png)).
- `cliente1` conserva su NTP por defecto.

Consecuencia: si pfSense se apaga, esos tres equipos pierden su fuente de hora.

## Problemas y rarezas

1. **DNS roto con la VPN encendida:** ver la sección 4.
2. **Prueba de NAT confusa:** filtrar los estados por `8.8.8.8` mostró tráfico DNS del CHR (TCP 53) y no mi ping, porque el CHR también usa esa IP. Lo resolví con una IP que nadie más usa (`9.9.9.9`) y filtrando con el ping aún en marcha: los estados de ICMP duran pocos segundos.
3. **Observación sin resolver:** el CHR consulta `8.8.8.8` por TCP 53. Causa no determinada.
4. **Redacción incompleta en la primera pasada:** el script no cubría `xmldata`, donde pfSense guarda las llaves SSH del servidor. La revisión por contenido lo detectó, corregí el patrón y subí el archivo solo cuando las dos revisiones salieron vacías. No llegó a subirse ninguna llave.
5. **La línea IPv6 del NAT** traduce `::1` a una dirección privada `fd17:…` de la WAN. No la investigué.

## Lo que aprendí

- Las reglas se leen de arriba hacia abajo, gana la primera coincidencia y lo demás se bloquea; una prueba de bloqueo solo vale si antes comprobé que sin la regla pasaba.
- Que un servicio escuche en todas las interfaces no significa que sea accesible: lo decide el firewall.
- El antilockout es una regla invisible que puede hacer parecer inútil a una regla de bloqueo.
- Una restauración se prueba cambiando algo y comprobando que vuelve, no solo subiendo el archivo.
- El estado del PC anfitrión (aquí, una VPN) puede romper el DNS de las VMs sin que la configuración de pfSense cambie.
- Un respaldo de pfSense guarda más secretos de los que parece (llaves SSH, certificado, hashes); se redacta y se revisa por contenido.

## Qué no se hizo

- Reglas de salida: la LAN conserva la regla de fábrica que permite todo hacia internet; no se restringió por puertos.
- IPv6: la regla de fábrica no se tocó y no se probó nada con IPv6.
- SSH solo con contraseña; no se configuró autenticación por llave pública.
- SSH al gateway desde la LAN con el antilockout activo: no probado (solo el puerto 443).
- DNS: causa del fallo original no determinada; no se averiguó cómo la VPN del anfitrión bloquea los DNS públicos;
- Sin experiencia en producción con pfSense.

## Referencias

- Netgate, [Outbound NAT](https://docs.netgate.com/pfsense/en/latest/nat/outbound-nat.html).
- Netgate, [Admin Access](https://docs.netgate.com/pfsense/en/latest/config/advanced-admin.html) (antilockout).
- Netgate, [Rule Methodology](https://docs.netgate.com/pfsense/en/latest/firewall/rule-methodology.html).
- Oracle, [Fine Tuning the VirtualBox NAT Engine](https://docs.oracle.com/en/virtualization/virtualbox/6.0/admin/changenat.html).
- NordVPN, [Change your DNS servers on Windows](https://support.nordvpn.com/hc/en-us/articles/19917312522257-Change-your-DNS-servers-on-Windows).

## Versiones y herramientas

- pfSense CE 2.9.1-RELEASE (amd64)
- VirtualBox
- Ubuntu Server (`librenms`, `glpi`, `cliente1`) y MikroTik RouterOS 7.25.1 (`chr`)
- PowerShell con OpenSSH
