# Laboratorio: GLPI 12, sistema de tickets e inventario, en VirtualBox

Fecha del laboratorio: 18/11/2026.
Resumen de lo que hice, paso a paso, para instalar GLPI y practicar el ciclo de un ticket de soporte.


> **Aviso de alcance (importante).** Este laboratorio es una **primera toma de contacto**, no experiencia en producción. Seguí una guía de instalación (computingforgeeks.com, "How To Install GLPI on Ubuntu 24.04") y comandos propuestos por un asistente de IA, y la mayor parte fue copiar, pegar y verificar el resultado. Los usuarios, el ticket y los artículos son de prueba, creados por mí en mi propia VM. No tengo experiencia usando un sistema de tickets en una empresa.
>
> **Nota de actualización (20/11/2026).** Este laboratorio describe la red como estaba el día que lo hice. Después rediseñé la red del proyecto Network Operations Lab y algunos datos de abajo ya no son los actuales: la VM ya no va en puente (hoy es `192.168.10.14` en `lab-core` y `192.168.56.14` de gestión); la RAM bajó de 2 GB a 1 GB; la zona horaria del sistema y de PHP ya es `America/Caracas`; y el límite de 3 VMs encendidas se descartó (caben 5). El texto original no se modificó, para conservar lo que hice en su momento. Estado actual: [arquitectura](../docs/arquitectura.md) y [direccionamiento](../docs/direccionamiento.md).
>
> **Nota de actualización (25/11/2026).** En la etapa de incidentes usé GLPI en tres incidentes simulados: tickets n.º 3, 4 y 5 (incidente-01, 02 y 03). Cada uno tuvo solución propuesta por `tecnico1`, aceptada por `usuario1`, y ticket cerrado. Añadí tres artículos a la base de conocimiento (seis en total). El n.º 4 lo abrió `usuario1` por un reporte; el n.º 3 y el n.º 5 los abrí como `tecnico1` a partir de una alerta. Comprobé la hora: GLPI muestra `America/Caracas` (ticket n.º 3 abierto a las 18:38:21 con mi PC en 18:39:34). El ticket n.º 2 no corresponde a ningún ejercicio documentado. Siguen sin configurarse entidades, categorías, SLA ni notificaciones.


---

## 1. Objetivo

Practicar con un sistema de gestión de tickets, algo que piden muchas ofertas de soporte IT (registro de solicitudes y resoluciones). Elegí **GLPI** (Gestionnaire Libre de Parc Informatique), un sistema libre de tickets e inventario.

## 2. Entorno

| Elemento | Detalle |
|---|---|
| PC anfitrión | Windows, Intel i7-2600K, 16 GB de RAM |
| Virtualización | VirtualBox |
| VM | `glpi`, Ubuntu Server 24.04 sin escritorio, **2 GB de RAM, 2 CPU**, adaptador en **puente** |
| Origen de la VM | Clon enlazado (linked clone) de un snapshot de otra VM Ubuntu que ya tenía (`full setup`) |
| Pila de software | Apache, PHP 8.3 (el de los repositorios de Ubuntu 24.04, sin PPA) y MariaDB |
| GLPI | Versión 12.0.2 (verificada con bin/console --version el 20/11/2026) |
| Acceso | SSH desde PowerShell con el usuario `robert`; interfaz web desde el navegador de Windows |
| Restricción | Máximo 3 VMs encendidas; la VM de LibreNMS se apagó mientras trabajaba en esta |

Instalé GLPI en una VM aparte y no en la de LibreNMS, porque esa ya usa Nginx y PHP 8.4, y se habrían pisado.

## 3. Pasos realizados

### 3.1 Crear la VM

1. Clon enlazado del snapshot `full setup`, nombre `glpi`, con **nuevas direcciones MAC**.
2. Configuración: 2048 MB de RAM, 2 CPU, Adaptador 1 en modo puente.
3. Cambié el nombre de la máquina y el archivo de hosts:

```
sudo hostnamectl set-hostname glpi
sudo sed -i 's/librenms/glpi/g' /etc/hosts
```

4. Con la VM apagada tomé un snapshot (`glpi-base-renombrada`, tomado antes de instalar nada). Los snapshots guardan también la configuración (RAM, CPU), por eso la cambié antes.

### 3.2 Problema de la IP duplicada

**Síntoma:** el clon recibió la misma IP (`192.168.1.102`) que la VM original de LibreNMS, aunque tenía MAC nueva.
**Diagnóstico:** comparé las MAC de ambas VMs en VirtualBox (eran distintas). Mi sospecha, **no verificada**, es que el router de casa identificó al clon por otro dato que heredó, el identificador que el cliente DHCP envía, que en Ubuntu puede derivarse del `machine-id`.
**Solución:** regenerar el `machine-id` del clon y reiniciar:

```
sudo rm -f /etc/machine-id
sudo systemd-machine-id-setup
sudo reboot
```

Después de reiniciar, la VM recibió una IP distinta `(no anoté cuál era; esa red ya no existe, ver la nota de actualización de arriba). Tomé un snapshot (`glpi with new ip`) antes de instalar GLPI.
**Lección:** dos VMs encendidas con la misma IP en la misma red causan conexiones intermitentes; una VM clonada puede necesitar un `machine-id` nuevo.

### 3.3 Acceso por SSH

Desde PowerShell: `ssh robert@<IP de glpi>`. Todos los comandos siguientes se ejecutaron por SSH.

### 3.4 Variables de la sesión

```
export GLPI_ROOT="/var/www/html/glpi"
export GLPI_DB="glpidb"
export GLPI_USER="glpiuser"
export GLPI_PASS="<contraseña-lab>"
```

(Solo para laboratorio; las contraseñas reales no se documentan aquí.)

### 3.5 Apache, PHP y MariaDB

```
sudo apt update
sudo apt install -y apache2 mariadb-server mariadb-client php php-cli php-common php-mbstring php-xml php-ldap php-intl php-gd php-mysql php-zip php-bcmath php-imap php-opcache php-curl libapache2-mod-php unzip wget curl
sudo systemctl enable --now apache2 mariadb
systemctl is-active apache2 mariadb
php --version
```

Comprobé que Apache y MariaDB estuvieran `active` y que PHP fuera 8.3.

### 3.6 Base de datos

Creé una base y un **usuario propio** para GLPI, en vez de usar `root` (principio de mínimo privilegio: si se filtra la contraseña de la aplicación, solo se accede a esa base):

```
sudo mariadb <<SQL
CREATE DATABASE ${GLPI_DB} CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
CREATE USER '${GLPI_USER}'@'localhost' IDENTIFIED BY '${GLPI_PASS}';
GRANT ALL PRIVILEGES ON ${GLPI_DB}.* TO '${GLPI_USER}'@'localhost';
FLUSH PRIVILEGES;
SQL
```

Zonas horarias (GLPI las usa para las fechas de los tickets):

```
sudo mariadb-tzinfo-to-sql /usr/share/zoneinfo | sudo mariadb mysql
sudo mariadb -e "GRANT SELECT ON mysql.time_zone_name TO '${GLPI_USER}'@'localhost'; FLUSH PRIVILEGES;"
```

### 3.7 Descargar GLPI

Obtuve la última versión publicada desde la API de GitHub, la descargué (unos 85 MB) y la descomprimí en `/var/www/html/`. Luego ajusté propietario y permisos:

```
sudo chown -R www-data:www-data "${GLPI_ROOT}"
sudo find "${GLPI_ROOT}" -type d -exec chmod 755 {} \;
sudo find "${GLPI_ROOT}" -type f -exec chmod 644 {} \;
sudo chmod -R +X "${GLPI_ROOT}/bin"
ls "${GLPI_ROOT}/public"
```

Verifiqué que existiera `index.php` en `public`.

### 3.8 Sitio de Apache

Creé `/etc/apache2/sites-available/glpi.conf` con un VirtualHost en el puerto 80 que apunta a `/var/www/html/glpi/public`, con `AllowOverride All` y `FallbackResource /index.php`. Luego:

```
sudo apache2ctl configtest
sudo a2dissite 000-default.conf
sudo a2ensite glpi.conf
sudo a2enmod rewrite
sudo systemctl reload apache2
```

`configtest` debe responder `Syntax OK`. Dejé solo ese sitio activo (el de Apache por defecto, desactivado). Sin HTTPS ni dominio real: es un laboratorio local.

### 3.9 Instalación de GLPI por consola

En vez del asistente web:

```
sudo -u www-data php "${GLPI_ROOT}/bin/console" db:install --db-name="${GLPI_DB}" --db-user="${GLPI_USER}" --db-password="${GLPI_PASS}" --default-language=es_ES --force --no-interaction --reconfigure
```

La interfaz quedó en español. Entré desde el navegador de Windows a `http://<IP de glpi>/`.

### 3.10 Cuentas por defecto

Según la guía, GLPI trae cuatro cuentas por defecto (`glpi`, `tech`, `normal`, `post-only`). **Cambié las contraseñas de las cuatro** desde *Administración > Usuarios*, porque GLPI muestra un aviso de seguridad hasta que se cambian. También se indicó borrar el instalador (`install/install.php`) `(el 20/11/2026 busqué con find y no existe ningún install.php fuera de vendor/; no sé si lo borré yo o si esta versión ya no lo trae)`.

## 4. Práctica con tickets

### 4.1 Usuarios de prueba

| Usuario | Perfil | Para qué |
|---|---|---|
| `usuario1` | Self-Service | Simula al usuario que abre el ticket |
| `tecnico1` | Technician | Simula al técnico que atiende |

### 4.2 Ciclo del ticket "PC sin internet" (ticket n.º 1)

1. Desde una ventana de incógnito, entré como `usuario1` y abrí el ticket **"PC Sin internet"**, con una descripción de prueba que simula a un usuario molesto.
2. Entré como `tecnico1`, me asigné el ticket y propuse una **solución**: *el cable del router está suelto, por favor conectarlo*.
3. Volví a `usuario1` y **acepté la solución**. GLPI registró la aceptación (la captura de ese día mostraba 2026-11-19 02:31 por `usuario1`; hoy GLPI muestra 2026-11-18 22:31 para el mismo evento, 4 horas menos, lo que cuadra con que antes marcaba UTC y ahora `America/Caracas`) y un seguimiento "Solución aprobada".

Con eso recorrí el flujo completo: **creación, asignación, solución propuesta, aprobación y cierre**, con el historial de cada paso visible en el propio ticket ([captura del historial](capturas/ticket-1-historial.png)).

### 4.3 Base de conocimiento

Escribí **3 artículos cortos** en la base de conocimiento, con los temas que me propusieron como ejemplo: PC sin internet, impresora que no imprime y cómo cambiar una contraseña. ([captura](capturas/glpi-articulos.png))

### 4.4 Control remoto

AnyDesk ya lo había probado antes, fuera de este laboratorio, por lo que no forma parte de lo documentado aquí.

### 4.5 Cierre

Apagué la VM y tomé el snapshot final **`glpi-listo`**.

---

## 5. Problemas encontrados

### 5.1 IP duplicada entre el clon y la VM original
Ver la sección 3.2. **Causa probable no verificada** (identificador DHCP derivado del `machine-id`); la solución (regenerar el `machine-id`) funcionó.

### 5.2 La VM "limpia" no era realmente limpia
El snapshot base (`full setup`) venía de la VM de LibreNMS: es un Ubuntu Server con SSH y con el nombre de máquina de aquella VM. Hubo que cambiar el hostname y regenerar el `machine-id` en cada clon. Los clones enlazados dependen de ese snapshot, así que no se puede borrar mientras existan.

### 5.3 Consola de VirtualBox poco práctica
Misma situación que en el laboratorio de MikroTik: teclado de EE. UU., sin mouse ni copiar y pegar. Todo el trabajo se hizo por SSH desde PowerShell, y la consola solo se usó para obtener la IP y activar el SSH.

---

## 6. Qué demuestra y qué no

**Qué sí:** instalé a mano una aplicación web con Apache, PHP y MariaDB; creé una base de datos con un usuario propio; entendí los pasos (servidor web, base de datos, permisos, sitio virtual); y recorrí con usuarios distintos el flujo de un ticket y la creación de documentación técnica en la base de conocimiento.

**Qué no:**
- Seguí una guía y comandos de una IA, copiando y pegando la mayor parte.
- Los datos son de prueba: el ticket del primer laboratorio, tres de incidentes simulados y seis artículos.
- No configuré entidades, categorías, SLA, notificaciones por correo, tareas programadas, HTTPS ni integración con directorio.
- No usé inventario (agentes) ni plugins.
- No tengo experiencia con un sistema de tickets en una empresa real.
