# Laboratorio: MikroTik CHR (RouterOS) con Winbox en VirtualBox

Fecha del laboratorio: 18/11/2026.
Resumen de lo que hice, paso a paso, en mi primer contacto con MikroTik RouterOS.

> **Alcance y transparencia:** laboratorio en VirtualBox con apoyo de IA (sin experiencia en producción). Se ejecutaron y verificaron los comandos paso a paso, documentando causas cuando se identificaron y marcando como "causa no determinada" o "no probado" lo que corresponda.
>
> **Estado actual (23/11/2026):** el CHR actúa como router interno (`192.168.10.12` en `lab-core`, gestión en `192.168.56.12`, DHCP y relevo DNS en `lab-clientes`), sin NAT ni reglas de red de casa (delegadas a pfSense), con `lab-limite` activo y SNMP v2c restringido a `192.168.10.13` para LibreNMS. El cuerpo de este documento conserva el montaje original; el detalle de cambios por fecha está en la **sección 8** al final y las referencias vigentes en [arquitectura](../docs/arquitectura.md), [direccionamiento](../docs/direccionamiento.md), [integración](../integracion/readme.md) y [librenms](../librenms/readme.md).

---

## 1. Objetivo

Familiarizarme con MikroTik RouterOS, un sistema muy usado por ISP y empresas pequeñas, usando mi base de Cisco (CCNA). La idea era comparar conceptos que ya conozco (IPs, DHCP, NAT, ACLs, VLANs) con la sintaxis de MikroTik.

## 2. Entorno

| Elemento | Detalle |
|---|---|
| PC anfitrión | Windows, Intel i7-2600K, 16 GB de RAM, unos 110 GB libres en C |
| Virtualización | VirtualBox (instalado en C) |
| Router | MikroTik **CHR** (Cloud Hosted Router), imagen `.vdi` de la página oficial de MikroTik |
| Herramientas de administración | Winbox (terminal y menús gráficos), SSH desde PowerShell |
| Cliente de prueba | VM Ubuntu Server 24.04 sin escritorio, llamada `cliente1` |
| Límite que me impuse | Máximo 3 VMs encendidas a la vez |

## 3. Topología

```
                 Internet
                    |
          Router de casa (192.168.1.0/24)
                    |
        PC con Winbox / PowerShell (192.168.1.140)
                    |
        [ether1]  CHR  192.168.1.123   (adaptador puente)
        [ether2]  10.10.10.1/24
        [vlan10 sobre ether2]  10.10.20.1/24
                    |
        Red interna de VirtualBox "lab-lan"
                    |
               cliente1  10.10.10.200  (por DHCP del CHR)
               enp0s3.10 10.10.20.2/24 (VLAN 10, creada a mano)
```

### Direccionamiento

| Equipo | Interfaz | Dirección | Cómo se obtuvo |
|---|---|---|---|
| CHR | ether1 | 192.168.1.123 | DHCP del router de casa |
| CHR | ether2 | 10.10.10.1/24 | Estática (la puse yo) |
| CHR | vlan10 | 10.10.20.1/24 | Estática (la puse yo) |
| cliente1 | enp0s3 | 10.10.10.200/24 | DHCP del CHR (pool 10.10.10.100-200) |
| cliente1 | enp0s3.10 | 10.10.20.2/24 | Estática, comandos `ip` (no persistente) |

## 4. Pasos realizados

### 4.1 Crear la VM del CHR

1. Descargué la imagen **VirtualBox (.vdi)** del CHR desde la página de descargas de MikroTik y la descomprimí.
2. En VirtualBox creé una VM nueva: nombre `mikrotik-chr`, tipo **Linux**, versión **Other Linux (64-bit)**, **1024 MB de RAM**, **1 CPU**, usando el `.vdi` como **disco existente** (no se crea disco nuevo).
3. Red: **Adaptador 1 en modo puente** (para que el CHR obtenga IP de mi router de casa y pueda llegar a internet).
4. Encendí la VM. Al arrancar preguntó si quería ver la licencia del software; respondí `n`.

### 4.2 Primer acceso

- Usuario `admin` con contraseña vacía; el sistema pide definir una contraseña nueva en el primer ingreso, y puse una de laboratorio.
- Comandos de reconocimiento:

```
/interface print
/ip address print
/ip dhcp-client print
/ping 8.8.8.8 count=4
/ip route print
```

- El CHR pidió IP por DHCP al router de casa y quedó con `192.168.1.123` en `ether1`. Tenía salida a internet (ping a 8.8.8.8 correcto).
- En la tabla de direcciones había dos columnas parecidas: **Address** (la IP del equipo, `192.168.1.123/24`) y **Network** (la red, `192.168.1.0`). Al principio confundí la segunda con la IP del CHR; ver el problema 6.1.

### 4.3 Winbox

- Descargué Winbox de la página de MikroTik, me conecté por IP (`192.168.1.123`) con `admin` y mi contraseña.
- Exploré los menús **IP > Addresses**, **IP > Routes** e **IP > Firewall**, comparándolos con lo que había visto en la consola.
- Usé **New Terminal** dentro de Winbox como terminal principal, porque la consola de VirtualBox me dio problemas (ver 4.4).

### 4.4 Problemas con la consola de VirtualBox

- **Teclado:** la consola del CHR usa la distribución de EE. UU., y no encontré un ajuste de idioma. Efecto: el símbolo `?` (que muestra las opciones de comandos) no se podía escribir bien; el Tab sí completaba comandos.
- **Mouse:** al hacer clic en la ventana, VirtualBox "captura" el mouse (desaparece) y se suelta con **Ctrl derecho**. No se puede copiar ni pegar.
- **Causa:** la VM no tiene Guest Additions y es un sistema en modo texto.
- **Solución:** dejar de usar la ventana de VirtualBox para escribir comandos y usar el **terminal de Winbox** (teclado de Windows, con copiar y pegar) o SSH desde PowerShell. La ventana de VirtualBox quedó solo para encender y apagar.

### 4.5 Red interna del laboratorio

1. Apagué el CHR y agregué un **Adaptador 2** en modo **Red interna** con nombre `lab-lan`. En RouterOS apareció como `ether2`.
2. Creé el cliente `cliente1`: **clon enlazado** (linked clone) de un snapshot de la VM Ubuntu que ya tenía (llamado `full setup`), con nuevas MAC, **1 GB de RAM** y su único adaptador en la red interna `lab-lan`.
3. En `cliente1` cambié el nombre de la máquina (`sudo hostnamectl set-hostname cliente1`). El prompt siguió mostrando el nombre viejo hasta cerrar sesión y volver a entrar; el cambio de nombre no afecta al usuario (`robert`).

### 4.6 IP, DHCP y NAT en el CHR

Comandos ejecutados en el terminal de Winbox:

```
/ip address add address=10.10.10.1/24 interface=ether2
/ip pool add name=pool-lab ranges=10.10.10.100-10.10.10.200
/ip dhcp-server add name=dhcp-lab interface=ether2 address-pool=pool-lab disabled=no
/ip dhcp-server network add address=10.10.10.0/24 gateway=10.10.10.1 dns-server=8.8.8.8
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

Qué hace cada uno: asigna la IP a `ether2`, define el rango del DHCP, crea el servidor DHCP en esa interfaz, le da gateway y DNS a los clientes, y activa NAT "masquerade" (el equivalente a NAT overload/PAT de Cisco) para que la red interna salga a internet con la IP de `ether1`.

**Comprobación:** `cliente1` recibió `10.10.10.200` por DHCP, con salida a internet (ping a `8.8.8.8` y a un nombre de dominio). El contador de la regla de masquerade subía (34 paquetes en una de las lecturas). Desde el CHR, `/ping 10.10.10.200 count=3` dio 3 de 3 respuestas, con latencia de unos 0,5 ms.

### 4.7 Firewall: que el laboratorio no vea mi red de casa

```
/ip firewall filter add chain=forward src-address=10.10.10.0/24 dst-address=192.168.1.0/24 action=drop comment="lab no ve la casa"
/ip firewall filter set 0 connection-state=new
```

- La regla bloquea lo que el laboratorio intente iniciar hacia mi red de casa.
- Al principio la creé **sin** `connection-state=new`. Sin esa condición, la regla también bloquea las **respuestas** del laboratorio hacia mi PC, por ejemplo las de un SSH que empiezo yo. Con `connection-state=new` solo se bloquean conexiones **nuevas** que empieza el laboratorio.
- Estado final verificado con `/ip firewall filter print`:

```
0  ;;; lab no ve la casa
   chain=forward action=drop connection-state=new src-address=10.10.10.0/24
   dst-address=192.168.1.0/24
```

### 4.8 Reenvío de puerto para SSH a cliente1 (problema sin resolver)

Quería entrar por SSH a `cliente1` desde mi PC, porque `cliente1` está en una red interna que Windows no ve. Creé un reenvío de puerto (`dst-nat`):

```
/ip firewall nat add chain=dst-nat in-interface=ether1 protocol=tcp dst-port=2222 action=dst-nat to-addresses=10.10.10.200 to-ports=22 comment="ssh a cliente1"
```

Y probé desde PowerShell: `ssh -p 2222 robert@192.168.1.123`. Resultado: **Connection refused**. Detalle completo del diagnóstico en el problema 6.2. Al final entré con un **servidor de salto** (jump host):

```
/ip ssh set forwarding-enabled=local
```
```
ssh -J admin@192.168.1.123 robert@10.10.10.200
```

El primer comando habilita en el CHR el reenvío de conexiones SSH; el segundo hace que mi PC se conecte primero al CHR y desde ahí a `cliente1`. Funcionó (pide la contraseña del CHR y luego la de `cliente1`). Alternativa que también existe: `/system ssh 10.10.10.200 user=robert` desde el terminal de Winbox.

### 4.9 VLAN 10

En el CHR:

```
/interface vlan add name=vlan10 interface=ether2 vlan-id=10
/ip address add address=10.10.20.1/24 interface=vlan10
```

En `cliente1` (interfaz física `enp0s3`):

```
sudo ip link add link enp0s3 name enp0s3.10 type vlan id 10
sudo ip addr add 10.10.20.2/24 dev enp0s3.10
sudo ip link set enp0s3.10 up
ping 10.10.20.1
```

Resultado: ping correcto a `10.10.20.1` (latencias de 0,3 a 0,7 ms). Observé que la red interna de VirtualBox **sí deja pasar** las tramas etiquetadas 802.1Q. Al pegar dos veces el primer comando salió `RTNETLINK answers: File exists`, que solo indica que la interfaz ya existía. Esa VLAN en `cliente1` no persiste al reiniciar, porque la creé con comandos sueltos y no con configuración de red.

**Cómo se ve en MikroTik:** menú *Interfaces > pestaña VLAN*, o `/interface vlan print` y `/ip address print`. No existe un `show vlan brief`: en este caso `vlan10` es una interfaz virtual montada sobre `ether2` (parecida a una subinterfaz `Gi0/0.10` de router-on-a-stick), no una entrada en una base de datos de VLANs. El tráfico **sin etiqueta** que entra por `ether2` lo procesa `ether2` (la red 10.10.10.0/24); `cliente1` tenía las dos IPs funcionando a la vez.

### 4.10 Control de velocidad, usuarios, servicios y respaldo

```
/queue simple add name=lab-limite target=10.10.10.0/24 max-limit=2M/5M
/user add name=operador group=read password=<contraseña-lab>
/ip service print
/ip service disable telnet
/export file=chr-lab
/system backup save name=chr-lab
```

- La *Simple Queue* limita el ancho de banda de la red del laboratorio (el formato de `max-limit` es subida/bajada). Revisé los contadores en Winbox, en el menú Queues.
- El usuario `operador` tiene grupo `read` (solo lectura), parecido al usuario de nivel 1 de mi proyecto de Cisco.
- Desactivé Telnet en `/ip service`, por seguridad.
- Hice un `export` (configuración en texto) y un `backup` (archivo binario).

### 4.11 Cierre

- Apagué el CHR con `/system shutdown` y `cliente1` con `sudo poweroff`.
- Con las VMs apagadas, tomé los snapshots **`chr-lab-listo`** (CHR) y **`chr-lab-listo`** (cliente1).

---

## 5. Equivalencias Cisco ↔ MikroTik que apunté

| Cisco | MikroTik |
|---|---|
| `show ...` | `print` (por ejemplo `show ip route` = `/ip route print`) |
| NAT overload / PAT | `masquerade` (usa la IP de la interfaz de salida) |
| NAT estático / port forward | `dst-nat` |
| ACL (sin estado) | `/ip firewall filter` (con estado si se usa `connection-state`) |
| Subinterfaz `Gi0/0.10` | Interfaz `vlan10` sobre `ether2` |
| VLAN nativa de un trunk | Tráfico sin etiqueta en la interfaz base (sin número de VLAN) |
| `show vlan brief` | No existe (`/interface vlan print`) |
| Usuario de nivel 1 | Grupo `read` |

Atajos: en el terminal de RouterOS, Tab completa comandos y `?` muestra opciones; los comandos están organizados por carpetas (`/ip`, `/interface`, `/system`).

---

## 6. Problemas encontrados

### 6.1 Confundí la dirección de red con la IP del equipo
**Síntoma:** al mirar *IP > Addresses* en Winbox, dije que `ether1` tenía `192.168.1.0/24`.
**Causa:** `192.168.1.0` era la columna **Network**; la IP del CHR está en **Address** (`192.168.1.123/24`).
**Lección:** en la tabla hay dos columnas parecidas; abrir la fila para ver la IP completa.

### 6.2 Reenvío de puerto: "Connection refused" (causa no determinada)
**Síntoma:** `ssh -p 2222 <usuario>@192.168.1.123` daba `Connection refused` con tres usuarios distintos. Pensé que era un problema de usuario, pero "refused" no es "Permission denied": significa que alguien contestó que ahí no escucha nadie, antes de llegar a pedir contraseña.

**Diagnóstico, de afuera hacia adentro:**
1. En `cliente1`, `systemctl status ssh` mostró el servicio **deshabilitado** (el paquete estaba instalado pero no activo). Lo activé con `sudo systemctl enable --now ssh`. Aun así seguía el rechazo.
2. `/ip firewall nat print` mostró la regla `dst-nat` bien escrita, y `/ip firewall nat print stats` mostró su contador de paquetes en **0** (nunca recibía tráfico).
3. `/ip firewall filter print` solo mostraba la regla "lab no ve la casa".
4. `/ping 10.10.10.200` desde el CHR respondió 3 de 3: el CHR sí llegaba a `cliente1`.
5. Con `/tool sniffer quick interface=ether1 port=2222` y un intento de SSH desde PowerShell, el CHR **recibía** los paquetes por `ether1` y respondía casi al instante (decenas de microsegundos), sin pasarlos a `cliente1`.
6. `Test-NetConnection 192.168.1.123 -Port 8291` (el puerto de Winbox) dio `TcpTestSucceeded: True`, así que `192.168.1.123` sí era el CHR.

**Causa:** no determinada. La regla estaba bien escrita, el paquete llegaba al CHR y el contador de la regla seguía en 0. Se propuso una regla temporal de log para ver el motivo; el estado final de la tabla NAT solo tiene las dos reglas (masquerade y `dst-nat`).
**Solución usada:** servidor de salto con `ssh -J` y `/ip ssh set forwarding-enabled=local` (sección 4.8).
**Lección:** distinguir "Connection refused" de "Permission denied" y de "timed out", y diagnosticar por tramos (¿llega el paquete?, ¿lo toma la regla?, ¿escucha el destino?).

### 6.3 Regla de filtro sin `connection-state=new`
**Síntoma (anticipado, no llegó a medirse en el laboratorio):** una regla `drop` de `10.10.10.0/24` hacia `192.168.1.0/24` sin estado también bloquearía las respuestas hacia mi PC.
**Solución:** `/ip firewall filter set 0 connection-state=new`, verificado con `print`.
**Lección:** es la diferencia entre filtrar sin estado (como una ACL) y con estado.

### 6.4 Nombre de la máquina distinto del prompt
**Síntoma:** tras `hostnamectl set-hostname cliente1`, el prompt seguía diciendo `robert@librenms`.
**Causa:** el prompt se actualiza al volver a iniciar sesión, y el cambio de nombre no modifica el usuario.

---

## 7. Qué demuestra y qué no

**Qué sí:** pasé de no conocer RouterOS a tener un CHR funcionando con DHCP, NAT, firewall, VLAN, limitación de ancho de banda, usuario de solo lectura, respaldo y acceso por Winbox y SSH, y entiendo qué hace cada comando que ejecuté y su equivalente en Cisco.

## 8. Historial de cambios posteriores al montaje original

- **20/11/2026 (rediseño de red):** `ether1` pasó de puente a `192.168.10.12` (`lab-core`); `lab-lan` pasó a `lab-clientes`; se añadió gestión en `ether3` (`192.168.56.12`); quedó obsoleta la regla hacia la red de casa y se descartó el tope de 3 VMs encendidas (5 VMs en el lab). Detalle en [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md).
- **22/11/2026 (integración):** se retiró el enmascaramiento NAT del CHR (queda en pfSense), se eliminaron el `dst-nat` del puerto 2222 y el filtro hacia la red de casa, se configuró relevo DNS (`allow-remote-requests=yes` y DNS `10.10.10.1` por DHCP en lugar de `8.8.8.8`), se dejó VLAN 10 persistente en `cliente1` con `netplan` y se conservó `lab-limite`. Detalle en [integración](../integracion/readme.md).
- **23/11/2026 (monitoreo):** se renombró la comunidad SNMP por defecto (`public`), se habilitó SNMP v2c solo para `192.168.10.13` y se integró el sondeo desde LibreNMS. Detalle y pruebas en [librenms](../librenms/readme.md).

**Qué no:**
- Es un laboratorio en VirtualBox, con un solo router y un solo cliente.
- Seguí guías y comandos propuestos por una IA, copiando y pegando la mayoría.
- No configuré bridge ni *bridge VLAN filtering*, enrutamiento dinámico, VPN, ni nada de un ISP real.
- La VLAN en `cliente1` no es persistente.
- No hay una causa determinada para el fallo del reenvío de puerto.
