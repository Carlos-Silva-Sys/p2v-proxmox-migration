```markdown
# 🖥️ p2v-proxmox-migration

[![Linux](https://img.shields.io/badge/Linux-FCC624?style=for-the-badge&logo=linux&logoColor=black)]()
[![Proxmox](https://img.shields.io/badge/Proxmox-E57000?style=for-the-badge&logo=proxmox&logoColor=white)]()
[![Bash](https://img.shields.io/badge/Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)]()
[![License](https://img.shields.io/badge/License-MIT-blue.svg)]()

> **Physical to Virtual Migration** - Migración profesional de servidores físicos Linux a máquinas virtuales en Proxmox VE

```
P2V MIGRATION - PROXMOX
Autor: Carlos Silva
Sistema: Debian / Proxmox VE
```

================================================================================

## 📋 Tabla de Contenidos

- [🎯 Descripción General](#-descripción-general)
- [📋 Requisitos Previos](#-requisitos-previos)
- [🖥️ En el Servidor Físico (Origen)](#️-en-el-servidor-físico-origen)
- [☁️ En el Nodo Proxmox (Destino)](#️-en-el-nodo-proxmox-destino)
- [🔧 Configuración Post-Migración](#-configuración-post-migración)
- [🚨 Solución de Problemas](#-solución-de-problemas)
- [📚 Comandos Rápidos](#-comandos-rápidos)
- [📞 Contacto](#-contacto)

================================================================================

## 🎯 Descripción General

Migración de servidor físico Linux a máquina virtual en Proxmox VE (P2V).
Este procedimiento está optimizado para ser **ESTABLE**, **REANUDABLE** (ante cortes de SSH)
y **PROFESIONAL**.

**Estrategia principal:**

| Método anterior (dd + gzip) | ✅ Método optimizado |
|-----------------------------|----------------------|
| Compresión en caliente falla | Backup SIN compresión (dd directo a RAW) |
| Si se corta SSH, se pierde el proceso | Todo comando largo dentro de `screen` |
| Múltiples conversiones manuales | Importación directa a QCOW2 con `qm importdisk` |

================================================================================

## 📋 Requisitos Previos

- Servidor físico con Linux (origen)
- Nodo Proxmox VE 7.x o 8.x (destino)
- Disco USB con capacidad **MAYOR o IGUAL** al disco del servidor origen
- Acceso SSH a ambos equipos
- Conocimientos básicos de Linux y virtualización

================================================================================

## 🖥️ En el Servidor Físico (Origen)

### 1️⃣ Preparar el USB

Conectar el USB y verificar su identificación:

```bash
lsblk
df -h
```

Asumiendo que el USB es `/dev/sdb1` con montaje en `/mnt/usb`:

```bash
mount /dev/sdb1 /mnt/usb
df -h /mnt/usb
```

### 2️⃣ Iniciar sesión SCREEN (protege contra cortes SSH)

```bash
screen -S backup_disco
```

> 💡 **Si se corta la conexión SSH:** `screen -r backup_disco` para reanudar  
> 🔓 **Para salir de screen sin detener el proceso:** `Ctrl + A` , luego `D`

### 3️⃣ Copiar el disco completo SIN compresión

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

### 4️⃣ Monitorear el progreso (desde otra terminal SSH)

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

### 5️⃣ Verificar el backup completado

Cuando el comando `dd` termine, verificar el tamaño del archivo:

```bash
ls -lh /mnt/usb/disco_sistema.raw
```

**Calcular el checksum (opcional, para verificar integridad):**

```bash
sha256sum /mnt/usb/disco_sistema.raw > /mnt/usb/disco_sistema.sha256
```

================================================================================

## ☁️ En el Nodo Proxmox (Destino)

### 6️⃣ Transferir la imagen al nodo Proxmox

**Opción A: USB físico conectado directamente al Proxmox** (recomendada)

```bash
# En el servidor Proxmox
lsblk
mount /dev/sdb1 /mnt/usb_backup
cp /mnt/usb_backup/disco_sistema.raw /var/lib/vz/images/
umount /mnt/usb_backup
```

**Opción B: Transferencia por red con netcat** (más rápida que scp)

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

**Opción C: SCP** (lento pero seguro y sencillo)

```bash
scp /mnt/usb/disco_sistema.raw root@<IP_PROXMOX>:/var/lib/vz/images/
```

### 7️⃣ Crear la VM en Proxmox (SIN disco)

```bash
# Crear la VM sin disco virtual
qm create 200 --name "servidor-migrado" --memory 4096 --cores 4 --bios seabios --net0 virtio,bridge=vmbr0

# Verificar que se creó sin discos
qm config 200
```

> **Nota:** No crear disco durante la creación de la VM. El disco se importará en el siguiente paso.

### 8️⃣ Importar y convertir RAW → QCOW2 en UN SOLO PASO

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

> **Si el proceso se interrumpe,** ejecutar nuevamente el mismo comando (sobrescribe el disco importado)

### 9️⃣ Attach el disco a la VM y configurar boot

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

### 🔟 Iniciar la VM y verificar

```bash
# Iniciar la VM
qm start 200

# Ver el estado
qm status 200

# Acceder a la consola (desde la web de Proxmox o terminal)
qm terminal 200
```

================================================================================

## 🔧 Configuración Post-Migración

### Si la VM no arranca (error de root=UUID)

1. En la consola de Proxmox, al ver el GRUB, presionar `e`
2. Buscar la línea que comienza con `linux`
3. Cambiar `root=UUID=...` por `root=/dev/vda1` (o `/dev/sda1` si usó SATA)
4. Presionar `Ctrl + X` para arrancar
5. Una vez dentro, actualizar GRUB:

```bash
update-grub
update-initramfs -u
```

### Configurar la red (la interfaz puede cambiar de nombre)

```bash
# Verificar el nombre de la interfaz
ip addr show

# Editar configuración de red
nano /etc/network/interfaces
```

**Ejemplo de configuración (Debian/Ubuntu):**

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

### Instalar QEMU Guest Agent

**Para Debian/Ubuntu:**

```bash
apt-get update
apt-get install qemu-guest-agent -y
systemctl enable qemu-guest-agent --now
```

**Para Red Hat/CentOS:**

```bash
yum install qemu-guest-agent -y
systemctl enable qemu-guest-agent --now
```

### Verificación final

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

## 🚨 Solución de Problemas Comunes

| Problema | Solución |
|----------|----------|
| **El proceso dd se interrumpió** | `dd if=/dev/sda bs=4M of=/mnt/usb/disco_sistema.raw seek=XX conv=notrunc` (XX = bloques ya copiados = tamaño_actual / 4M) |
| **La VM arranca pero no encuentra discos LVM** | Dentro de la VM: `pvscan && vgchange -ay && update-initramfs -u` |
| **La interfaz de red cambió de eth0 a ens18** | Editar `/etc/network/interfaces` con el nuevo nombre o usar `net.ifnames=0` en GRUB |
| **El disco RAW no cabe en el storage de Proxmox** | `gzip /var/lib/vz/images/disco_sistema.raw` y luego `qm importdisk` desde el `.gz` (más lento pero más pequeño) |

================================================================================

## 📚 Comandos Rápidos para Monitoreo

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
