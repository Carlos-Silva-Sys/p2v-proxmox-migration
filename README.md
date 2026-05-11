```markdown
# p2v-proxmox-migration

Migracion de Servidor fisico a Maquina Virtual

```
P2V MIGRATION - PROXMOX
Autor: Carlos Silva
Sistema: Debian / Proxmox VE

================================================================================

DESCRIPCIÓN
================================================================================

Migración de servidor físico Linux a máquina virtual en Proxmox VE (P2V).
Este procedimiento está optimizado para ser ESTABLE, REANUDABLE (ante cortes de SSH)
y PROFESIONAL.

ESTRATEGIA PRINCIPAL:
- Backup SIN compresión (dd directo a RAW) → más estable y rápido
- Todo comando largo dentro de screen → protegido contra desconexiones
- Importación directa a QCOW2 con qm importdisk → un solo paso sin conversiones manuales

================================================================================

REQUISITOS PREVIOS
================================================================================

- Servidor físico con Linux (origen)
- Nodo Proxmox VE (destino)
- Disco USB con capacidad MAYOR o IGUAL al disco del servidor origen
- Acceso SSH a ambos equipos
- Conocimientos básicos de Linux y virtualización

================================================================================

PASO 1: PREPARAR EL USB EN EL SERVIDOR FÍSICO
================================================================================

Conectar el USB y verificar su identificación:

```bash
lsblk
df -h
```

Asumiendo que el USB es /dev/sdb1 con montaje en /mnt/usb:

```bash
mount /dev/sdb1 /mnt/usb
df -h /mnt/usb
```

================================================================================

PASO 2: INICIAR SESIÓN SCREEN (PROTEGE CONTRA CORTES SSH)
================================================================================

```bash
screen -S backup_disco
```

**Si se corta la conexión SSH**, volver a entrar con:

```bash
screen -r backup_disco
```

**Para salir de screen sin detener el proceso:** `Ctrl + A` , luego `D`

================================================================================

PASO 3: COPIAR EL DISCO COMPLETO SIN COMPRESIÓN
================================================================================

**DENTRO de la sesión screen**, ejecutar:

```bash
dd if=/dev/sda bs=4M of=/mnt/usb/disco_sistema.raw status=progress
```

**Explicación:**
- `if=/dev/sda` : disco origen (cambiar según el servidor)
- `bs=4M` : bloque de 4 MB (óptimo para rendimiento)
- `of=` : archivo de salida en el USB
- `status=progress` : muestra progreso (en sistemas modernos)

**Para servidores antiguos** (Debian 6, RHEL 5, etc.) que no soportan `status=progress`:

```bash
dd if=/dev/sda bs=4M of=/mnt/usb/disco_sistema.raw
# Luego en otra terminal monitorear con:
watch -n 10 'kill -USR1 $(pgrep dd)'
```

**💡 TIP: Si el sistema tiene `pv` instalado** (Pipe Viewer), puedes usarlo para una barra de progreso más visual sin comprimir:

```bash
dd if=/dev/sda bs=4M | pv > /mnt/usb/disco_sistema.raw
```

**Tamaño final esperado:** Igual al tamaño del disco origen (ej: 80 GB, 500 GB, etc.)
**Tiempo estimado:** ~30-50 minutos por cada 100 GB (depende de la velocidad del USB)

================================================================================

PASO 4: MONITOREAR EL PROGRESO (DESDE OTRA TERMINAL SSH)
================================================================================

**Ver el tamaño actual del archivo cada 10 segundos:**

```bash
watch -n 10 'ls -lh /mnt/usb/disco_sistema.raw'
```

**Verificar que el proceso dd sigue corriendo:**

```bash
ps aux | grep dd | grep -v grep
```

**Ver las sesiones screen activas:**

```bash
screen -ls
```

**Entrar a una sesión screen en curso:**

```bash
screen -r backup_disco
```

================================================================================

PASO 5: VERIFICAR EL BACKUP COMPLETADO
================================================================================

Cuando el comando `dd` termine, verificar el tamaño del archivo:

```bash
ls -lh /mnt/usb/disco_sistema.raw
```

**Calcular el checksum (opcional, para verificar integridad):**

```bash
sha256sum /mnt/usb/disco_sistema.raw > /mnt/usb/disco_sistema.sha256
```

================================================================================

PASO 6: TRANSFERIR LA IMAGEN AL NODO PROXMOX
================================================================================

**Opción A: USB físico conectado directamente al Proxmox**

```bash
# En el servidor Proxmox
lsblk
mount /dev/sdb1 /mnt/usb_backup
cp /mnt/usb_backup/disco_sistema.raw /var/lib/vz/images/
umount /mnt/usb_backup
```

**Opción B: Transferencia por red con netcat (más rápida que scp)**

> ⚠️ **ADVERTENCIA:** `netcat` transmite los datos sin encriptar. Úsalo SOLO en redes de confianza (red local, VPN, etc.). Para Internet o redes inseguras, usa SCP.

```bash
# En el nodo Proxmox (receptor)
screen -S recibir_raw
nc -l -p 9000 > /var/lib/vz/images/disco_sistema.raw
# Ctrl+A, D para salir

# En el servidor origen (emisor)
cd /mnt/usb
screen -S enviar_raw
nc <IP_DEL_PROXMOX> 9000 < disco_sistema.raw
# Ctrl+A, D para salir
```

**Opción C: SCP (lento pero seguro y sencillo)**

```bash
scp /mnt/usb/disco_sistema.raw root@<IP_PROXMOX>:/var/lib/vz/images/
```

================================================================================

PASO 7: CREAR LA VM EN PROXMOX (SIN DISCO)
================================================================================

```bash
# Crear la VM sin disco virtual
qm create 200 --name "servidor-migrado" --memory 4096 --cores 4 --bios seabios --net0 virtio,bridge=vmbr0

# Verificar que se creó sin discos
qm config 200
```

**Nota:** No crear disco durante la creación de la VM. El disco se importará en el siguiente paso.

================================================================================

PASO 8: IMPORTAR Y CONVERTIR RAW → QCOW2 EN UN SOLO PASO
================================================================================

**Este es el comando clave del método optimizado:**

```bash
qm importdisk 200 /var/lib/vz/images/disco_sistema.raw local --format qcow2
```

**Explicación:**
- `200` : ID de la VM
- `/var/lib/vz/images/disco_sistema.raw` : ruta del archivo RAW
- `local` : storage de Proxmox (puede ser `local-lvm` u otro)
- `--format qcow2` : convierte a QCOW2 durante la importación

**Tiempo estimado:** Similar al tiempo de copia original (~30-50 min por 100 GB)

**Si el proceso se interrumpe,** ejecutar nuevamente el mismo comando (sobrescribe el disco importado)

================================================================================

PASO 9: ATTACH EL DISCO A LA VM Y CONFIGURAR BOOT
================================================================================

```bash
# Verificar que el disco se importó (aparece como unused0)
qm config 200

# Attach el disco como VirtIO (recomendado para rendimiento)
qm set 200 --virtio0 local:200/vm-200-disk-0.qcow2

# Configurar orden de boot
qm set 200 --boot order=virtio0

# Habilitar QEMU Guest Agent (opcional pero recomendado)
qm set 200 --agent enabled=1
```

**Para servidores antiguos que no tienen drivers VirtIO:** usar `--sata0` en lugar de `--virtio0`

```bash
qm set 200 --sata0 local:200/vm-200-disk-0.qcow2
```

================================================================================

PASO 10: INICIAR LA VM Y VERIFICAR
================================================================================

```bash
# Iniciar la VM
qm start 200

# Ver el estado
qm status 200

# Acceder a la consola (desde la web de Proxmox o terminal)
qm terminal 200
```

================================================================================

PASO 11: CONFIGURACIÓN DENTRO DE LA VM (PRIMER ARRANQUE)
================================================================================

**Si la VM no arranca (error de root=UUID):**

1. En la consola de Proxmox, al ver el GRUB, presionar `e`
2. Buscar la línea que comienza con `linux`
3. Cambiar `root=UUID=...` por `root=/dev/vda1` (o `/dev/sda1` si usó SATA)
4. Presionar `Ctrl + X` para arrancar
5. Una vez dentro, actualizar GRUB:

```bash
update-grub
update-initramfs -u
```

**Configurar la red (la interfaz puede cambiar de nombre):**

```bash
# Verificar el nombre de la interfaz
ip addr show

# Editar configuración de red
nano /etc/network/interfaces
```

Ejemplo de configuración (Debian/Ubuntu):

```bash
auto lo
iface lo inet loopback

auto ens18
iface ens18 inet static
    address 192.168.1.100
    netmask 255.255.255.0
    gateway 192.168.1.1
    dns-nameservers 8.8.8.8
```

**Instalar QEMU Guest Agent:**

```bash
apt-get update
apt-get install qemu-guest-agent
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent
```

**Para sistemas Red Hat/CentOS:**

```bash
yum install qemu-guest-agent
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent
```

================================================================================

PASO 12: VERIFICACIÓN FINAL
================================================================================

Dentro de la VM migrada, verificar:

```bash
# Discos montados
df -h

# Red funcionando
ip addr show
ping -c 4 8.8.8.8

# Procesos y servicios
ps aux
systemctl status  # o service --status-all

# Logs sin errores críticos
dmesg | grep -i error
```

================================================================================

SOLUCIÓN DE PROBLEMAS COMUNES
================================================================================

**Problema 1: El proceso dd se interrumpió**

```bash
# Verificar si el archivo RAW tiene tamaño parcial
ls -lh /mnt/usb/disco_sistema.raw

# Reanudar dd con seek (omite lo ya copiado)
dd if=/dev/sda bs=4M of=/mnt/usb/disco_sistema.raw seek=XX conv=notrunc
# (XX = bloques ya copiados = tamaño_actual / 4M)
```

**Problema 2: La VM arranca pero no encuentra discos LVM**

```bash
# Dentro de la VM
pvscan
vgchange -ay
mount /dev/mapper/vg0-root /mnt
# Luego actualizar initramfs
update-initramfs -u
```

**Problema 3: La interfaz de red cambió de eth0 a ens18**

```bash
# Solución 1: Renombrar interfaz en /etc/network/interfaces
# Solución 2: Configurar GRUB para mantener nombres tradicionales
echo 'GRUB_CMDLINE_LINUX="net.ifnames=0 biosdevname=0"' >> /etc/default/grub
update-grub
```

**Problema 4: El disco RAW no cabe en el storage de Proxmox**

```bash
# Ver espacio disponible en Proxmox
df -h /var/lib/vz/

# Si no hay suficiente espacio, usar compresión post-backup
gzip /var/lib/vz/images/disco_sistema.raw
# Luego qm importdisk desde el .gz (más lento pero más pequeño)
```

================================================================================

RESUMEN DE COMANDOS ÚTILES PARA MONITOREO
================================================================================

```bash
# Ver tamaños de backups
ls -lh /mnt/usb/*.raw

# Ver procesos activos
ps aux | grep -E "dd|screen" | grep -v grep

# Ver sesiones screen
screen -ls

# Reingresar a una sesión
screen -r backup_disco

# Matar una sesión screen si es necesario
screen -X -S backup_disco quit

# Monitorear progreso de dd en sistemas antiguos
watch -n 10 'kill -USR1 $(pgrep dd)'

# Calcular tiempo estimado de finalización
# (tamaño_actual / velocidad_promedio)
```


## 📝 Autor

Carlos Silva  
GitHub: [@Carlos-Silva-Sys](https://github.com/Carlos-Silva-Sys)
