# p2v-proxmox-migration
Migracion de Servidor fisico a Maquina Virtual

```
P2V MIGRATION - PROXMOX
Autor: Carlos Silva
Sistema: Debian / Proxmox VE

================================================================================

DESCRIPCIÓN
================================================================================

Migración de servidor físico a máquina virtual en Proxmox (P2V).
Este procedimiento fue probado y funciona en entornos reales.

================================================================================

PASO 1: CREAR IMAGEN DEL SERVIDOR FÍSICO
================================================================================

1. Conectar disco externo USB al servidor físico
2. Verificar montaje:
   lsblk
   df -h

(Supongamos que está montado en /mnt/usb)

3. Crear imagen comprimida del disco:
   dd if=/dev/sda bs=4M | pv | gzip > /mnt/usb/servidor.img.gz

- if=/dev/sda: disco físico completo
- pv: muestra progreso
- gzip: comprime la imagen

================================================================================

PASO 2: SUBIR IMAGEN AL NODO PROXMOX
================================================================================

Desde tu máquina local o el servidor físico:

scp /mnt/usb/servidor.img.gz root@IP_DEL_NODO:/var/lib/vz/images/

Verificar:
ls -lh /var/lib/vz/images/servidor.img.gz

================================================================================

PASO 3: CREAR VM EN PROXMOX
================================================================================

En la interfaz web de Proxmox:
1. Crear nueva VM (ejemplo: ID 9000)
2. Asignar CPU, RAM y disco virtual
3. Tipo de disco: SATA
4. Tamaño del disco: igual o mayor que el original
5. NO instalar sistema operativo
6. Apagar la VM: qm stop 9000

================================================================================

PASO 4: PREPARAR EL DISCO VIRTUAL
================================================================================

Ubicar el disco virtual:
ls /var/lib/vz/images/9000/

(Verás algo como vm-9000-disk-0.qcow2)

Renombrar y crear disco en formato RAW:
mv vm-9000-disk-0.qcow2 vm-9000-disk-0.qcow2.bak
qemu-img create -f raw /var/lib/vz/images/9000/vm-9000-disk-0.raw 500G

================================================================================

PASO 5: RESTAURAR IMAGEN EN DISCO VIRTUAL
================================================================================

Ejecutar desde el shell del nodo Proxmox:

gunzip -c /var/lib/vz/images/servidor.img.gz | dd of=/var/lib/vz/images/9000/vm-9000-disk-0.raw bs=4M status=progress

Verificar progreso:
watch -n 5 ls -lh /var/lib/vz/images/9000/vm-9000-disk-0.raw

================================================================================

PASO 6: SI SE CANCELÓ EL PROCESO
================================================================================

Eliminar el .qcow2 incompleto:
rm /var/lib/vz/images/9000/vm-9000-disk-0.qcow2

================================================================================

PASO 7: CONVERTIR A QCOW2 (OPCIONAL)
================================================================================

Si prefieres usar QCOW2:

qemu-img convert -p -f raw -O qcow2 /var/lib/vz/images/9000/vm-9000-disk-0.raw /var/lib/vz/images/9000/vm-9000-disk-0.qcow2

Luego borrar el RAW:
rm vm-9000-disk-0.raw

================================================================================

PASO 8: INICIAR Y VERIFICAR LA VM
================================================================================

Iniciar la VM:
qm start 9000

Si no arranca:
- En GRUB, presiona 'e'
- Cambia root=UUID=... por root=/dev/sda1
- Presiona Ctrl + X para arrancar

Verificar red:
- La interfaz puede cambiar (eth0 → ens18)
- Ajustar /etc/network/interfaces o Netplan

================================================================================

PASO 9: INSTALAR QEMU GUEST AGENT
================================================================================

apt-get install qemu-guest-agent
systemctl enable qemu-guest-agent
systemctl start qemu-guest-agent

================================================================================

CONTACTO
================================================================================
GitHub: CarlosSilva32d-blip
Correo: carlossilva32d@gmail.com

LICENCIA: Uso libre para fines educativos y profesionales.
```
