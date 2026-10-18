# Procedimiento estándar: actualización de Proxmox VE 8 a Proxmox VE 9

> **Tipo:** procedimiento operativo  
> **Idioma:** español  
> **Ámbito:** nodos Proxmox VE independientes o pertenecientes a un clúster  
> **Destino:** Debian 13 (Trixie) y Proxmox VE 9  
> **Estado:** guía genérica. Adaptar a la topología y consultar la documentación oficial de la versión antes de ejecutarla.

## 1. Objetivo

Documentar una actualización controlada de Proxmox VE 8 a Proxmox VE 9, incluyendo la preparación del nodo, las dependencias de máquinas virtuales y contenedores, los kernels fijados, la migración de repositorios, el reinicio y la validación posterior.

Este documento complementa, pero no sustituye, la guía oficial de actualización de Proxmox.

## 2. Reglas de seguridad

- Actualizar **un solo nodo cada vez**.
- No continuar si hay errores sin entender en las comprobaciones previas.
- Disponer de copias de seguridad verificadas y acceso alternativo a consola (IPMI/KVM o consola local), especialmente si el nodo aloja red o cortafuegos.
- No reiniciar un nodo que proporciona almacenamiento compartido sin conocer las dependencias de los demás nodos.
- No ejecutar reparaciones destructivas sobre discos o sistemas de archivos sin identificar el dispositivo y tener copia de seguridad.
- No copiar configuraciones específicas de un nodo a otro sin revisarlas.
- En un clúster, comprobar quorum y estado de todos los nodos antes y después.

## 3. Criterios para detenerse

Detén el procedimiento si:
- La versión de origen no está dentro de una ruta de actualización admitida.
- `pve8to9 --full` presenta errores sin resolver.
- El clúster no tiene quorum o hay nodos desconectados inesperadamente.
- No hay copias de seguridad recientes y verificadas.
- No está claro qué discos o montajes utiliza una VM/LXC.
- El nodo aloja el único NAS, almacenamiento o cortafuegos necesario para recuperar la infraestructura y no existe un plan de contingencia probado.
- APT propone eliminar componentes esenciales o presenta conflictos sin diagnosticar.
- No hay acceso alternativo a consola y el acceso remoto podría perderse.

## 4. Inventario previo

Ejecuta los comandos como `root` o con privilegios equivalentes y guarda las salidas en el registro de intervención.

### 4.1 Identificar nodo y versiones

```bash
hostname
pveversion -v
cat /etc/os-release
uname -r
uptime
```

Anota el nombre del nodo, las versiones de Proxmox VE y Debian, el kernel activo y la hora de inicio.

### 4.2 Clúster y quorum

En un nodo de clúster:

```bash
pvecm status
pvecm nodes
```

Confirma que hay quorum y que los nodos esperados aparecen conectados. En un nodo independiente estos comandos pueden no ser aplicables.

### 4.3 Inventario de VM y LXC

```bash
qm list
pct list
```

Para cada invitado crítico, guarda su configuración:

```bash
qm config <VMID>
pct config <CTID>
```

Sustituye los marcadores por los identificadores reales. Documenta la función del invitado, criticidad, almacenamiento, dependencias y forma de validar que funciona tras el reinicio.

### 4.4 Discos, sistemas de archivos y almacenamiento

```bash
lsblk -f
findmnt
cat /etc/fstab
pvesm status
cat /etc/pve/storage.cfg
```

Revisa configuraciones de VM/LXC y localiza:
- Discos virtuales alojados en NAS/NFS.
- Discos físicos pasados a una VM.
- Controladores PCI/USB pasados a una VM.
- Montajes de host pasados a LXC (`mp0`, `mp1`, etc.).
- Bind mounts, rutas adicionales y dependencias de `/etc/fstab`.
- Unidades systemd o scripts que monten discos al arrancar.

Si se pasan discos físicos, registra los identificadores estables utilizados, por ejemplo `/dev/disk/by-id/`. No des por hecho que un dispositivo conservará el mismo nombre `/dev/sdX` tras reiniciar.

**No desconectes, formatees ni modifiques particiones durante la actualización.** Si no está claro qué controla un dispositivo, investígalo antes.

### 4.5 NAS y NFS

```bash
findmnt -t nfs,nfs4
grep -R -nE 'nfs|nfs4' /etc/fstab /etc/pve/storage.cfg 2>/dev/null
```

Registra el ID de almacenamiento, servidor, exportación, punto de montaje y qué invitados dependen de cada recurso. Si el NAS es una VM/LXC del propio clúster, identifica el orden de parada y arranque.

### 4.6 Kernel fijado (*pinned*) y paquetes retenidos

```bash
uname -r
proxmox-boot-tool kernel list
proxmox-boot-tool status
apt-mark showhold
```

Distingue dos mecanismos:

1. **Kernel fijado mediante Proxmox:** revisar `proxmox-boot-tool kernel list` y, si existe, `/etc/kernel/proxmox-boot-pin`.
2. **Paquetes retenidos por APT:** revisar `apt-mark showhold`.

No desancles ni elimines nada automáticamente. Averigua por qué está fijado, qué kernel se está ejecutando y qué kernels alternativos hay disponibles. Conserva una alternativa conocida siempre que sea viable.

Solo si se ha decidido quitar el pin y se ha comprobado el método de arranque:

```bash
proxmox-boot-tool kernel unpin
```

Este comando quita la selección fija de kernel; no borra kernels antiguos ni libera paquetes retenidos por APT. No uses `apt-mark unhold` indiscriminadamente.

Comprueba también la configuración de arranque local antes de aceptar reemplazos:

```bash
cat /etc/default/grub
```

Conserva los parámetros específicos de hardware, almacenamiento o red que sigan siendo necesarios. No copies este archivo desde otro host.

### 4.7 Red y servicios fallidos

```bash
ip -br address
ip route
systemctl --failed
```

Documenta interfaces, bridges, VLAN, rutas estáticas, DNS, VPN y servicios críticos. Si el nodo aloja un firewall/router virtual, prepara acceso fuera de banda y un plan de recuperación probado.

## 5. Copias de seguridad y preparación de invitados

1. Realiza copias de seguridad recientes de las VM/LXC críticos.
2. Verifica que el almacenamiento de copias está accesible y que las tareas han terminado correctamente.
3. Si es posible, prueba una restauración en un destino aislado.
4. Guarda fuera del disco a actualizar una copia de configuraciones importantes, como `/etc/pve/`, `/etc/network/interfaces`, `/etc/fstab`, `/etc/default/grub`, `/etc/apt/` y los archivos de `/etc/kernel/` relacionados con el arranque.
5. No restaures indiscriminadamente datos internos de `/var/lib/pve-cluster` sobre un clúster activo. Sigue el procedimiento oficial de recuperación.

### 5.1 Si hay un NAS en una VM o LXC

- Identifica todos los consumidores del NAS antes de detenerlo.
- Comprueba que no haya copias, transferencias o escrituras importantes en curso.
- Detén ordenadamente los invitados que dependen de ese NAS o migra sus servicios a un destino independiente.
- Detén el servicio NAS y la VM/LXC de forma ordenada.
- No fuerces el apagado salvo que el invitado esté bloqueado y se haya evaluado el riesgo.

Si el NAS es el único destino de copias de seguridad, no actualices sin disponer de una copia independiente. Si el NAS reside en el nodo a actualizar, planifica la ventana teniendo en cuenta que los otros nodos pueden perder acceso a ese almacenamiento.

### 5.2 Discos físicos y montajes pasados a invitados

Registra si cada disco se presenta como disco virtual, disco físico, controlador passthrough o montaje del host dentro de un LXC. Confirma que los datos están sincronizados y que los sistemas de archivos se han desmontado limpiamente cuando corresponda.

No desmontes del host un sistema de archivos que siga utilizándose desde un contenedor. Tras actualizar, comprueba que los dispositivos y rutas siguen presentes antes de iniciar el invitado.

### 5.3 Parada ordenada

Usa los identificadores reales y solo después de confirmar el plan:

```bash
qm shutdown <VMID>
pct shutdown <CTID>
```

Comprueba el resultado con `qm list` y `pct list`. `qm stop` y `pct stop` no deben ser el método habitual; son opciones de último recurso si el invitado no responde.

## 6. Repositorios

Consulta la guía oficial para la versión concreta y el tipo de suscripción. No mezcles repositorios de Debian 12 Bookworm con Debian 13 Trixie ni repositorios de Proxmox VE 8 con Proxmox VE 9.

### 6.1 Copia y revisión de APT

```bash
cp -a /etc/apt "/root/apt-backup-$(date +%F-%H%M%S)"
grep -R -nE 'bookworm|bullseye|trixie|pve-enterprise|pve-no-subscription' /etc/apt/sources.list /etc/apt/sources.list.d 2>/dev/null
```

Revisa los archivos `.list` y `.sources`. Adapta los repositorios oficiales de Debian y Proxmox según la documentación de destino. Para repositorios de terceros, confirma primero que el proveedor admite Debian 13/Trixie y la versión de Proxmox instalada.

No cambies automáticamente la suite de repositorios de terceros. Si un proveedor utiliza deliberadamente una suite propia como `stable`, conserva esa configuración hasta confirmar sus instrucciones de compatibilidad.

### 6.2 Actualizar índices

```bash
apt update
```

No continúes si hay errores de firma, repositorios duplicados, suites inexistentes o índices que no se han descargado.

### 6.3 Comprobación previa

```bash
pve8to9 --full
```

Lee todos los avisos y errores. Resuelve los errores antes de continuar y documenta las advertencias que se hayan revisado y aceptado. La salida varía según la versión; no ignores avisos por haber visto otros parecidos en una actualización anterior.

## 7. Actualizar el sistema

En una ventana de mantenimiento y con las copias verificadas:

```bash
apt update
apt full-upgrade
```

Revisa el plan de paquetes antes de confirmar. Si APT propone eliminar Proxmox VE, paquetes esenciales, almacenamiento o componentes de red que no entiendes, cancela y analiza el motivo.

### 7.1 Preguntas sobre archivos de configuración

No existe una respuesta universal a los avisos de APT:
- **Mantener la versión local:** conserva personalizaciones, pero puede dejar fuera ajustes nuevos del paquete.
- **Instalar la versión del mantenedor:** puede sobrescribir personalizaciones.

Compara las diferencias y conserva manualmente los ajustes necesarios. Presta especial atención a `/etc/default/grub`, la configuración de red y los servicios personalizados. Si no sabes qué elegir, no respondas a ciegas: guarda la versión local y revisa el diff que ofrece el sistema.

## 8. Reinicio y validación

Cuando APT termine sin errores, comprueba la versión y los kernels instalados:

```bash
pveversion -v
proxmox-boot-tool kernel list
```

Planifica el reinicio y asegúrate de disponer de acceso alternativo:

```bash
reboot
```

Después del arranque:

```bash
uptime
uname -r
pveversion -v
systemctl --failed
ip -br address
ip route
pvesm status
```

En un clúster:

```bash
pvecm status
pvecm nodes
```

Confirma que:
- El kernel activo es el esperado.
- El nodo está en línea y el clúster conserva quorum.
- Red, bridges, VLAN, rutas y acceso de administración funcionan.
- Los servicios esenciales están activos.
- Los almacenamientos esperados aparecen activos.
- Los discos, dispositivos passthrough y montajes se detectan correctamente.

No inicies todos los invitados a la vez si dependen de un NAS, DNS, VPN u otros servicios que todavía no estén operativos.

## 9. Recuperar NFS si aparece `Stale file handle`

### 9.1 Síntomas

Un montaje NFS puede aparecer montado, pero fallar al acceder con:

```text
Stale file handle
```

Esto suele indicar que el cliente conserva una referencia a un objeto remoto que ya no es válida, por ejemplo tras reiniciar el servidor NFS o cambiar una exportación. No demuestra por sí solo que haya pérdida de datos.

### 9.2 Identificar el almacenamiento

```bash
pvesm status
findmnt -t nfs,nfs4
```

Identifica el ID de almacenamiento de Proxmox, el punto de montaje, el servidor y la exportación. Revisa `/etc/pve/storage.cfg` y comprueba en el NAS que la exportación sigue existiendo.

No uses `ping` como única prueba: ICMP puede estar bloqueado aunque NFS funcione. Comprueba la conectividad y los servicios con métodos apropiados para la red.

### 9.3 Desmontaje normal

Antes de desmontar, confirma que no hay tareas de copia, VM, LXC ni procesos que estén utilizando el almacenamiento. No desmontes un almacenamiento que contenga discos activos de invitados.

Si es seguro, desmonta el punto exacto identificado con `findmnt`:

```bash
umount /punto/de/montaje
```

Sustituye el marcador por la ruta real. Si indica que está ocupado, identifica qué lo utiliza; no fuerces el desmontaje a ciegas.

### 9.4 Reactivar y verificar

```bash
pvesm status
findmnt -t nfs,nfs4
```

Proxmox puede volver a montar el almacenamiento cuando se consulta o se utiliza. Comprueba que el ID aparece activo, que el origen NFS es correcto y que se puede listar el contenido. Haz una prueba de lectura inocua antes de devolverlo a producción.

Si persiste el error:
1. Comprueba que la exportación existe y que la ruta es correcta en el NAS.
2. Revisa conectividad, firewall, versión NFS y permisos.
3. Consulta los registros del cliente y del servidor.
4. Confirma que no quedan clientes usando una exportación antigua.
5. No borres ni recrees la definición del almacenamiento como primer intento.
6. Evita `umount -l`, `umount -f` o reinicios forzados hasta entender las dependencias y el riesgo.

Repite las comprobaciones en **cada nodo** con un montaje afectado. Reparar el montaje en un nodo no repara automáticamente los montajes de los demás.

## 10. Volver a iniciar invitados

Cuando red, almacenamiento y clúster estén verificados:

1. Inicia primero los servicios de infraestructura necesarios (NAS, DNS, VPN, etc.) según el diseño de la instalación.
2. Verifica montajes NAS y discos físicos.
3. Comprueba que las exportaciones y recursos compartidos están disponibles.
4. Inicia el resto de VM/LXC por grupos.
5. Revisa las tareas de Proxmox y los registros de los invitados.

No inicies una VM con discos dependientes de NFS mientras el almacenamiento esté inactivo. Si el nodo aloja el firewall principal, sigue el plan específico de recuperación y verifica la conectividad desde un cliente.

## 11. Validación funcional final

La interfaz web no es suficiente. Comprueba:
- Acceso web y SSH al host.
- Quorum del clúster.
- Lectura de cada almacenamiento.
- Lectura/escritura controlada en recursos compartidos, si procede.
- Arranque y funcionamiento de VM/LXC críticos.
- Red, VLAN, DNS, VPN y servicios publicados.
- Copias de seguridad y tareas programadas.
- Dispositivos passthrough y montajes de invitados.
- Registros del sistema y tareas fallidas.

No hagas pruebas de escritura sobre repositorios de copias o sistemas sensibles sin definir una ruta de prueba segura.

## 12. Pasar al siguiente nodo

Antes de continuar:
- Confirma que el nodo actualizado está estable.
- Verifica que el clúster conserva quorum.
- Comprueba que las copias siguen disponibles.
- Confirma que no quedan montajes NFS obsoletos.
- Verifica los servicios compartidos y la conectividad.

Ten especial cuidado con los nodos que alojan NAS, almacenamiento compartido o firewall principal. Deja para el final los nodos cuya caída pueda impedir recuperar la infraestructura, salvo que exista una contingencia probada.

## 13. Registro de intervención

| Campo | Valor |
|---|---|
| Fecha y hora | |
| Técnico responsable | |
| Nodo | |
| Versión inicial de Proxmox/Debian | |
| Versión final de Proxmox/Debian | |
| Kernel antes/después | |
| Resultado de `pve8to9 --full` | |
| Repositorios de terceros revisados | |
| Copias verificadas | |
| VM/LXC detenidos | |
| Discos passthrough y montajes revisados | |
| Almacenamientos NFS comprobados | |
| Incidencias y resolución | |
| Validación final | |

## 14. Referencias oficiales

- Guía de actualización de Proxmox VE: <https://pve.proxmox.com/wiki/Upgrade_from_8_to_9>
- Documentación de Proxmox VE: <https://pve.proxmox.com/pve-docs/>

La documentación oficial y las notas de publicación de la versión concreta tienen prioridad si algún paso de este documento difiere de ellas.

## Lista de comprobación rápida

- [ ] Ruta de actualización admitida y ventana de mantenimiento confirmada.
- [ ] Copias de seguridad verificadas y fuera del nodo.
- [ ] Quorum del clúster correcto.
- [ ] VM/LXC, NAS, discos passthrough y montajes inventariados.
- [ ] Dependencias NFS conocidas.
- [ ] Kernel activo, pinning y paquetes retenidos revisados.
- [ ] Repositorios oficiales y de terceros revisados.
- [ ] `apt update` sin errores.
- [ ] `pve8to9 --full` revisado y errores resueltos.
- [ ] `apt full-upgrade` completado sin conflictos pendientes.
- [ ] Reinicio completado con el kernel esperado.
- [ ] Red, quorum, almacenamiento, VM/LXC y servicios verificados.
- [ ] NFS comprobado en cada nodo afectado.
- [ ] Registro de intervención completado.

