# Orden de encendido y apagado

Fecha: 20/11/2026

Las 5 VMs caben a la vez en el PC anfitrión (i7-2600K, 16 GB). Ver `arquitectura.md`, sección 8.

---

## 1. Encendido

Esperar a que cada VM termine de arrancar antes de encender la siguiente. Cada una depende de la anterior.

| Paso | VM | Por qué en este orden | Comprobar antes de seguir |
|---|---|---|---|
| 1 | `pfsense` | Da DHCP, NAT y DNS a `lab-core` | La consola muestra WAN `10.0.2.15` y LAN `192.168.10.1`; abre `https://192.168.56.11` |
| 2 | `chr` | Es el gateway de los clientes y reparte DHCP en `lab-clientes` | Conecta por Winbox o SSH a `192.168.56.12`; `/ip address print` muestra `192.168.10.12` en `ether1` |
| 3 | `librenms` | Necesita DHCP de pfSense | `ip -br a`: `.13` en `enp0s3`; `http://192.168.56.13` abre |
| 4 | `glpi` | Necesita DHCP de pfSense | `ip -br a`: `.14` en `enp0s3`; `http://192.168.56.14` abre |
| 5 | `cliente1` | Necesita el DHCP del CHR, y el CHR necesita a pfSense para salir | `ping -c 3 192.168.10.1` y `ping -c 3 8.8.8.8` responden |

Tiempos medidos el 20/11/2026 con cronómetro, desde el clic en Iniciar hasta poder entrar. Son aproximados (incluyen el tiempo de escribir usuario y clave):

| VM | Tiempo | Hasta cuándo se midió |
|---|---|---|
| `pfsense` | 1:55 | Pantalla de login web (incluye aceptar el aviso del certificado) |
| `chr` | 31 s | Entrar por SSH |
| `librenms` | 37 s | Login en la consola |
| `glpi` | 40 s | Login en la consola |
| `cliente1` | 38 s | Login en la consola |

Si `cliente1` se enciende antes que el CHR o pfSense, Ubuntu puede mostrar "Failed to connect to changelogs.ubuntu.com" al entrar. No es un error, solo que aún no hay salida a internet.

## 2. Revisión rápida después de encender

En las tres Ubuntu:

```
systemctl --failed
```

Lo normal es `0 loaded units listed`. Además, en `librenms` y `glpi`:

```
systemctl is-active mariadb
```

Debe decir `active`.

## 3. Apagado

Orden inverso, siempre con apagado limpio:

1. `cliente1`, `glpi`, `librenms`: `sudo poweroff`.
2. `chr`: `/system shutdown`.
3. `pfsense`: Diagnostics → Halt System.

Evitar cerrar la ventana de VirtualBox con "apagar la máquina", porque equivale a desenchufarla.

## 4. Después de un apagón

El 20/11/2026 se fue la luz con las 5 VMs encendidas y arrancaron sin problemas. Aun así, al volver:

1. Encender en el orden de la sección 1. Es normal que VirtualBox las marque como "Abortada" y que pfSense tarde más.
2. Hacer la revisión de la sección 2.
3. No restaurar snapshots a ciegas: si algo falla, anotar el mensaje exacto.

## 5. Snapshots

Tomarlos con la VM apagada. No restaurar los viejos: devuelven los adaptadores a como estaban antes del rediseño. No borrar ninguno: hay clones enlazados.

| VM | Vigente | Anteriores (se conservan) |
|---|---|---|
| `pfsense` | `diseno-listo` | `gestion-lista`, `pfsense-base-listo` |
| `chr` | `diseno-listo` | `gestion-lista`, `chr-lab-listo` |
| `librenms` | `diseno-listo` | `gestion-lista`, "Antes de ejercicios" |
| `glpi` | `diseno-listo` | `gestion-lista`, `glpi-listo` |
| `cliente1` | `plantilla-base` | `cliente2-plantilla-base` |
| `cliente-origen` | no se toca | n/a |

Para clonar un cliente desde `plantilla-base`: generar MAC nuevas en todos los adaptadores, renombrar el host y regenerar el `machine-id`.

Snapshot del chat 4: `integrado-base`.
