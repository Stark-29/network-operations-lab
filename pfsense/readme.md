# pfSense CE 2.9.1 en VirtualBox: primer laboratorio

Instalación y configuración básica de un firewall/router pfSense en una VM de VirtualBox, con una LAN de laboratorio aislada y una VM cliente (`cliente1`) que sale a internet a través de él. Este laboratorio es una pieza de un proyecto mayor que integrará MikroTik CHR, LibreNMS, GLPI y pfSense.

> **Nota de transparencia:** seguí este laboratorio con instrucciones paso a paso de una IA (asistente de chat). No es experiencia en producción: es mi **primera toma de contacto con pfSense**. Lo documenté para entender cada pieza y poder explicarla. No he operado pfSense en una red real.

> **Nota de actualización (20/11/2026).** Este laboratorio describe la red como estaba el día que lo hice. Después rediseñé la red del proyecto Network Operations Lab y algunos datos de abajo ya no son los actuales: la LAN de pfSense pasó de `lab-lan` a `lab-core`; `cliente1` ya no cuelga de pfSense sino del CHR; la RAM de la VM bajó de 2 GB a 1 GB; la web se administra por `https://192.168.56.11`, sin túnel SSH; y ya existen reservas DHCP. El texto original no se modificó, para conservar lo que hice en su momento. Estado actual: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md).

**Fecha:** 19/11/2026

---

## Resumen

- pfSense CE **2.9.1-RELEASE** (amd64), instalado con el **Netgate Installer v1.2** en una VM de VirtualBox.
- **WAN** (Wide Area Network) en un adaptador NAT; **LAN** (Local Area Network) en una red interna llamada `lab-lan`.
- Servidor DHCP en la LAN: `192.168.10.100` a `192.168.10.150`.
- `cliente1` (Ubuntu Server) sale a internet por pfSense. Verificado con `ping` a una IP y a un nombre de dominio.
- Acceso a la interfaz web desde mi PC mediante un túnel SSH a través de `cliente1`, porque la `lab-lan` no es visible desde el anfitrión.
- Un problema de DNS (Domain Name System) resuelto activando el modo de reenvío del DNS Resolver.

---

## Entorno

| Elemento | Valor |
|---|---|
| Hipervisor | VirtualBox |
| pfSense | CE 2.9.1-RELEASE amd64 (build 20261117-1822) |
| Instalador | `netgate-installer-v1.2-RELEASE-amd64.iso` |
| Recursos de la VM | 2 GB de RAM, 2 CPU, disco virtual de 20 GB |
| Disco | ZFS (Zettabyte File System), Stripe con 1 disco, esquema GPT (`ada0`) |
| Adaptador 1 | NAT, usado como WAN (`em0`, MAC `0800278C57D7`) |
| Adaptador 2 | Red interna `lab-lan`, usado como LAN (`em1`, MAC `080027230BCD`) |
| Hostname por defecto | `pfSense.home.arpa` |

## Topología

```
              Internet
                 │
        [ NAT de VirtualBox ]  10.0.2.0/24
                 │
          em0 (WAN) 10.0.2.15
            ┌────┴─────┐
            │ pfSense  │
            └────┬─────┘
          em1 (LAN) 192.168.10.1/24
                 │
        [ lab-lan: red interna ]
                 │
      enp0s3  192.168.10.100 (DHCP)
            ┌────┴─────┐
            │ cliente1 │   Ubuntu Server (solo terminal)
            └────┬─────┘
      enp0s8  192.168.56.101 (solo-anfitrión)
                 │
        PC anfitrión (túnel SSH hacia pfSense)
```

## Plan de direcciones

| Red / equipo | Dirección | Notas |
|---|---|---|
| WAN de pfSense | `10.0.2.15/24` | Por DHCP, desde el NAT de VirtualBox |
| LAN de pfSense | `192.168.10.1/24` | IP fija; también gateway y DNS de la LAN |
| Pool DHCP de la LAN | `192.168.10.100` – `192.168.10.150` | |
| `cliente1` (enp0s3) | `192.168.10.100/24` | Por DHCP de pfSense |
| `cliente1` (enp0s8) | `192.168.56.101/24` | Red solo-anfitrión de VirtualBox |

Se eligió `192.168.10.0/24` para mantener el laboratorio separado de la red de casa y de la red del laboratorio de MikroTik (`10.10.10.0/24`).

---

## Pasos realizados

### 1. Instalación con el Netgate Installer

1. Arranqué la VM desde la ISO del instalador.
2. El instalador **solo pide configurar la WAN**, porque necesita internet para descargar pfSense: seleccioné la interfaz del adaptador NAT, modo **DHCP client**, **VLAN tagging desactivado**.
3. Elegí pfSense CE y la versión estable. El instalador ofreció 2.9.1 (había anotado 2.9.0 al planear).
4. Disco: ZFS, Stripe con 1 disco, GPT, seleccionando `ada0` (20 GB).
5. Reinicié y quité la ISO de la unidad óptica de VirtualBox.

Tras el primer arranque, la consola mostró la WAN (`em0`) con `10.0.2.15/24` y la LAN (`em1`) con `192.168.10.1/24`.

### 2. DHCP de la LAN

La IP de la LAN ya aparecía configurada después de la instalación. Pasé igualmente por la opción **2) Set interface(s) IP address** de la consola, con la LAN en IP fija (respondí `n` a "configurar por DHCP"), y dejé el servidor DHCP activo en `192.168.10.100` – `192.168.10.150`. No verifiqué cuál de los dos pasos activó el servidor DHCP.

`cliente1` recibió `192.168.10.100`, lo que confirmó que el DHCP funciona.

### 3. Segundo adaptador en `cliente1` (acceso desde mi PC)

La `lab-lan` es una red interna de VirtualBox: mi PC no puede llegar a `192.168.10.1`. Como `cliente1` es solo terminal y no tiene las Guest Additions (no se puede copiar y pegar en su consola), le agregué un segundo adaptador **solo-anfitrión** (Host-Only) para entrar por SSH desde PowerShell.

1. Con la VM apagada: Configuración → Red → segundo adaptador → **Adaptador solo-anfitrión**.
2. Al arrancar, Ubuntu Server no configuró el adaptador nuevo (`enp0s8` en DOWN, sin IP). Hay que declararlo en netplan.
3. Archivo `/etc/netplan/50-cloud-init.yaml`:

```yaml
network:
  version: 2
  ethernets:
    enp0s3:
      dhcp4: true
    enp0s8:
      dhcp4: true
      dhcp4-overrides:
        use-routes: false
```

`use-routes: false` evita que el adaptador solo-anfitrión añada una ruta por defecto: la salida a internet debe seguir yendo por pfSense (`enp0s3`).

4. `sudo netplan apply` y `ip -br a`. `enp0s8` tomó `192.168.56.101`.

### 4. Acceso a la interfaz web por túnel SSH

Desde PowerShell en mi PC:

```
ssh -L 8443:192.168.10.1:443 robert@192.168.56.101
```

Con la sesión abierta, en el navegador: `https://localhost:8443`. El navegador/antivirus avisa de un certificado no confiable; es normal, porque pfSense usa un certificado autofirmado. El mensaje de protección contra "DNS rebind" que a veces aparece al entrar por `localhost` **no apareció** en mi caso.

Credenciales por defecto de pfSense: usuario `admin`, contraseña `pfsense`; la contraseña se cambió en el wizard.

### 5. Asistente inicial (wizard)

| Pantalla | Valor |
|---|---|
| General Information | Hostname `pfSense`; DNS `1.1.1.1` y `8.8.8.8` |
| Time Server | NTP (Network Time Protocol) por defecto `2.pfsense.pool.ntp.org`; zona horaria UTC-4 (Venezuela) |
| WAN | Tipo DHCP; **desmarcada** "Block RFC1918 Private Networks" (la WAN recibe una IP privada del NAT de VirtualBox, `10.0.2.15`, y se bloquearía); "Block bogon networks" se dejó marcada |
| LAN | Sin cambios (`192.168.10.1/24`). Cambiar esta IP cortaría el túnel SSH |
| Admin password | Cambiada (credencial de laboratorio) |

### 6. Verificación desde `cliente1`

```
ip route
ping -c 3 8.8.8.8
ping -c 3 google.com
```

- `ip route`: `default via 192.168.10.1 dev enp0s3`. La salida a internet pasa por pfSense.
- `ping 8.8.8.8`: 3 de 3 respuestas, 0 % de pérdida, ~39 ms de media. La ruta y el NAT (Network Address Translation) funcionan.
- `ping google.com`: **falló** al principio con `Temporary failure in name resolution`. Ver el problema 1 abajo. Después de la corrección, respondió.

---

## Problemas encontrados y cómo se resolvieron

### 1. El DNS no resolvía nombres, pero el ping a una IP sí funcionaba
**Síntoma:** `ping 8.8.8.8` respondía y `ping google.com` daba `Temporary failure in name resolution`.
**Diagnóstico:** `resolvectl status enp0s3` mostró que `cliente1` usaba `192.168.10.1` como único DNS (el DNS Resolver de pfSense, que usa Unbound) y el dominio `home.arpa`. El fallo estaba, por tanto, en el resolvedor de pfSense, no en el cliente ni en el DHCP.
**Solución:** Services → DNS Resolver → General Settings → activar **Enable Forwarding Mode** → Save y Apply Changes. En ese modo, pfSense reenvía las consultas a los DNS configurados (`1.1.1.1` y `8.8.8.8`) en lugar de resolver desde los servidores raíz. Después, `ping google.com` funcionó.
**Causa:** no determinada. Sospecha no verificada: la resolución desde la raíz fallaba detrás del NAT de VirtualBox o de mi conexión.
**Nota:** el dashboard de pfSense mostraba además dos DNS desconocidos (`103.86.96.100` y `103.86.99.100`). No determiné su origen.

### 2. El adaptador nuevo de `cliente1` quedó sin IP
**Síntoma:** `enp0s8` en DOWN y sin dirección.
**Causa:** Ubuntu Server solo configura las interfaces declaradas en netplan; la nueva no estaba.
**Solución:** agregarla en netplan (ver paso 3).

### 3. Error de netplan: `expected mapping (check indentation)`
**Síntoma:** `sudo netplan apply` dio un error de indentación.
**Causa:** en el YAML, las claves de `enp0s8` (`dhcp4`, `dhcp4-overrides`) estaban alineadas con el nombre de la interfaz en vez de más a la derecha, y el nombre se había escrito `enpos8` (con la letra "o") en lugar de `enp0s8` (con cero).
**Solución:** corregir la indentación (4 espacios para `enp0s8`, 6 para sus claves, 8 para `use-routes`) y escribir el nombre con cero. Usar espacios, nunca tabuladores.
**Lección:** en YAML la indentación es parte de la sintaxis.

---

## Cosas que aprendí de este primer contacto

- El Netgate Installer de pfSense CE solo configura la WAN durante la instalación; las interfaces se terminan de asignar en el primer arranque, desde el menú de consola.
- La mayor parte de la configuración de pfSense se hace por la interfaz web; la consola sirve para el arranque inicial.
- La lista de DNS del dashboard de pfSense no es lo que usa un cliente: el cliente usa lo que pfSense le entrega por DHCP.
- Una red interna de VirtualBox está aislada del anfitrión; para administrar pfSense hace falta un camino adicional (aquí, un túnel SSH por una VM con segundo adaptador).

## Respaldo y snapshots

- Snapshot de VirtualBox de la VM de pfSense apagada, con la configuración base ya lista.
- Snapshot de `cliente1` con sus dos adaptadores.
- Configuración exportada desde Diagnostics → Backup & Restore (`config.xml`). **Contiene contraseñas y hashes**: antes de subirlo a GitHub, reemplazarlos por `<REDACTED>` o no subirlo.

Para clonar `cliente1` y simular otro cliente: usar la opción de VirtualBox de generar **nuevas direcciones MAC** en todos los adaptadores, y en el clon renombrar el host y regenerar el `machine-id` (si no, DHCP puede tratarlo como el mismo equipo).

---

## Qué no se hizo (pendiente)

- Reglas de firewall de la LAN: se usan las que trae pfSense por defecto. No se probaron ni se editaron.
- SSH habilitado en el propio pfSense (opción 14 de la consola o System → Advanced).
- SNMP (Simple Network Management Protocol) para que LibreNMS monitoree pfSense.
- Reservas DHCP (IP fija por MAC) para las VMs del laboratorio.
- Conexión con los otros laboratorios (MikroTik CHR, LibreNMS, GLPI).
- Pruebas de reinicio, restauración del respaldo y VLANs.
- Sin experiencia en producción con pfSense.

## Datos a revisar antes de publicar

- Nombre de usuario de `cliente1` (`robert`) y contraseñas de laboratorio.
- `config.xml`: hashes y contraseñas.
- MACs y direcciones del laboratorio (son virtuales, pero decidir si se dejan).
- Los DNS desconocidos (`103.86.x.x`) que aparecieron en el dashboard.

## Versiones y herramientas

- pfSense CE 2.9.1-RELEASE (amd64)
- Netgate Installer v1.2-RELEASE
- VirtualBox
- Ubuntu Server (`cliente1`), con netplan
- PowerShell con OpenSSH para el túnel