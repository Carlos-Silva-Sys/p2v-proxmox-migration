```markdown
# 🖥️ p2v-proxmox-migration

*Physical to Virtual Migration*

> Migración profesional de servidores físicos Linux a máquinas virtuales en Proxmox VE. Sin compresión, con screen, y un solo comando para importar.

## ⚠️ IMPORTANTE - ANTES DE EMPEZAR

- El disco USB debe tener **más espacio** que el disco del servidor origen
- Todos los comandos largos deben ejecutarse dentro de `screen` para evitar pérdida por corte SSH
- Cambiar `/dev/sda` por el disco correcto del servidor (ej: `/dev/cciss/c0d0` en HP CCISS)

## 📋 Descripción

**Problema:** Los backups con compresión (`dd | gzip`) fallan o se congelan en hardware antiguo, además de que un corte SSH mata el proceso.

**Solución:** Backup directo SIN compresión (`dd` a RAW) + `screen` para proteger el proceso + `qm importdisk` para convertir a QCOW2 en un solo paso.

## 🚀 Tecnologías

| Tecnología | Versión | Puerto |
|------------|---------|--------|
| Debian (origen) | 6+ | - |
| Proxmox VE (destino) | 7.x / 8.x | 8006 |
| Screen | cualquier | - |
| dd | cualquier | - |

## ⚙️ Procedimiento completo

### 1. En el servidor físico (origen)
```

```bash
# Montar USB
mount /dev/sdb1 /mnt/usb

# Iniciar screen
screen -S backup_disco

# Copiar disco SIN compresión
dd if=/dev/sda bs=4M of=/mnt/usb/disco_sistema.raw status=progress

# Salir de screen sin detener: Ctrl+A, D
```

**Monitorear desde otra terminal:**

```bash
watch -n 10 'ls -lh /mnt/usb/disco_sistema.raw'
screen -ls
screen -r backup_disco
```

### 2. Transferir al nodo Proxmox

```bash
# Opción USB directo (recomendada)
mount /dev/sdb1 /mnt/usb_backup
cp /mnt/usb_backup/disco_sistema.raw /var/lib/vz/images/

# Opción netcat (solo red local de confianza)
# En Proxmox: nc -l -p 9000 > /var/lib/vz/images/disco_sistema.raw
# En origen: nc <IP_PROXMOX> 9000 < /mnt/usb/disco_sistema.raw
```

### 3. En Proxmox - crear VM e importar

```bash
# Crear VM SIN disco
qm create 200 --name "servidor-migrado" --memory 4096 --cores 4 --net0 virtio,bridge=vmbr0

# Importar RAW a QCOW2 (comando clave)
qm importdisk 200 /var/lib/vz/images/disco_sistema.raw local --format qcow2

# Attach disco y configurar boot
qm set 200 --virtio0 local:200/vm-200-disk-0.qcow2
qm set 200 --boot order=virtio0
qm set 200 --agent enabled=1

# Iniciar
qm start 200
qm terminal 200
```

## 🔧 Pasos después de ejecutar (dentro de la VM)

**Si no arranca (error root=UUID):**

1. En GRUB presionar `e`
2. Cambiar `root=UUID=...` por `root=/dev/vda1`
3. `Ctrl+X` para arrancar
4. Luego: `update-grub && update-initramfs -u`

**Configurar red:**

```bash
# Ver nueva interfaz (probablemente ens18)
ip addr show

# Editar /etc/network/interfaces con la IP original del servidor
nano /etc/network/interfaces
```

**Instalar QEMU Guest Agent:**

```bash
apt-get update && apt-get install qemu-guest-agent -y
systemctl enable qemu-guest-agent --now
```

## 👤 Autor

**Carlos Silva**  
GitHub: [Carlos-Silva-Sys](https://github.com/Carlos-Silva-Sys)
```
