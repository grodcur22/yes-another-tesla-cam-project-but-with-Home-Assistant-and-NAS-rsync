# yes-another-tesla-cam-project-but-with-Home-Assistant-and-NAS-rsync

# TeslaCam Pi Zero 2W Auto-Sync

A smart, automated solution for Tesla Dashcam and Sentry Mode backups using a Raspberry Pi Zero 2W. It emulates a USB drive for the car while automatically syncing footage to a remote NAS via Tailscale and providing real-time status updates to Home Assistant via MQTT.

## 🚀 Features

- **USB Emulation:** Emulates a mass storage device for Tesla's V9+ Dashcam system.
- **Remote Sync:** Uses rsync over a secure Tailscale tunnel to offload footage.
- **Smart Scheduling:** Syncs immediately when connected to home WiFi; restricts to a 3:00 AM window when on the car's cellular hotspot.
- **Home Assistant Integration:** Real-time status tracking via MQTT (Syncing, Online, Backup OK, Paused).
- **Safety First:** Respects "Read-Only" locks when the Tesla is actively writing to avoid filesystem corruption.

## 🛠️ Prerequisites

- **Hardware:** Raspberry Pi Zero 2W (with a high-end microSD card)
- **Network:** Tailscale account for secure Mesh VPN
- **Smart Home:** Home Assistant with an MQTT Broker (Mosquitto)
- **Remote Storage:** A NAS or Server with SSH access

## 🔧 Installation & Configuration

### 1. Hardware Setup (USB Gadget Mode)

Modify the boot configuration to enable the `dwc2` USB driver.

**/boot/firmware/config.txt**

```plaintext
[cm4]
dtoverlay=dwc2,dr_mode=peripheral

[all]
dtoverlay=dwc2,dr_mode=peripheral
```

**/boot/firmware/cmdline.txt**

Add the following at the end of the line:

```plaintext
modules-load=dwc2,g_mass_storage
```

### 2. Create the Virtual Disk

Create a blank container file to act as the USB drive.

```bash
sudo dd if=/dev/zero of=/teslacam.bin bs=1M count=32000
sudo mkfs.vfat /teslacam.bin -F 32 -n TESLACAM
```

### 3. SSH Key Authentication

Allow the Pi to sync to your NAS without a password.

```bash
ssh-keygen -t rsa -b 4096
ssh-copy-id root@<YOUR_NAS_TAILSCALE_IP>
```

## 📜 The Sync Script

Create the script at `/home/teslacam/sync_tesla.sh`.

```bash
#!/bin/bash
# Evitar que el script se ejecute si ya hay uno corriendo
if pidof -x $(basename $0) > /dev/null; then
  for pid in $(pidof -x $(basename $0)); do
    if [ $pid != $$ ]; then
      exit 0
    fi
  done
fi

# Configuración MQTT
MQTT_HOST="XXX.XXX.XXX.XXX"
MQTT_USER="********"
MQTT_PASS="********"
TOPIC_STATUS="tesla/pi/status"
TOPIC_TIME="tesla/pi/last_sync"

# --- WATCHDOG DEL BINARIO ---
if [ ! -f "/teslacam.bin" ]; then
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_STATUS -m "ERROR: Binario no encontrado. Recreando..."
    # Recrear el binario (usamos 24000 para mantener tus 24GB)
    sudo dd if=/dev/zero of=/teslacam.bin bs=1M count=24000
    sudo mkfs.vfat /teslacam.bin -F 32 -n TESLACAM
    
    # Recrear estructura interna
    sudo mkdir -p /mnt/tesladisk
    sudo mount -o loop /teslacam.bin /mnt/tesladisk
    sudo mkdir -p /mnt/tesladisk/TeslaCam
    touch /mnt/tesladisk/TeslaCam/.mcu1_ready
    sudo umount /mnt/tesladisk
    
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_STATUS -m "Binario restaurado OK"
fi

# 1. Detectar red y hora
CURRENT_SSID=$(/usr/sbin/iwgetid -r)
CURRENT_HOUR=$(date +%-H)
CURRENT_MIN=$(date +%-M)

# Pausa si detecta la red WiFi del coche (evitar bucles de datos)
if [ "$CURRENT_SSID" == "Tesla" ]; then
    if [ "$CURRENT_HOUR" -ne 3 ] || [ "$CURRENT_MIN" -gt 30 ]; then
        mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_STATUS -m "Pausado (Red Tesla)"
        exit 0
    fi
fi

# --- MEJORA 1: Reparación del Contenedor (Anti-Corrupción) ---
# Si el Tesla cortó la energía, esto repara el binario antes de intentar montarlo
sudo dosfsck -a /teslacam.bin

mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_STATUS -m "Sincronizando..."

# --- PASO 1: Montaje Seguro ---
sync
sudo umount -l /mnt/tesladisk 2>/dev/null
# Mantenemos tu offset intacto
sudo mount -t vfat -o loop,ro,umask=000,time_offset=-480 /teslacam.bin /mnt/tesladisk

# --- PASO 2: Verificación y Auto-reparación (Anti-formateo) ---
# Si el coche formateó el disco, recreamos la carpeta TeslaCam
if [ -d "/mnt/tesladisk" ] && [ ! -d "/mnt/tesladisk/TeslaCam" ]; then
    echo "Carpeta TeslaCam no encontrada. Recreando estructura..."
    sudo mount -o remount,rw /mnt/tesladisk
    sudo mkdir -p /mnt/tesladisk/TeslaCam
    # Tip para MCU1: crear archivo vacío para asegurar que el sistema lo reconozca
    touch /mnt/tesladisk/TeslaCam/.mcu1_ready
    sudo mount -o remount,ro /mnt/tesladisk
fi

# --- PASO 3: Sincronización ---
if [ -d "/mnt/tesladisk/TeslaCam" ]; then
    # rsync hacia el NAS
    rsync -av /mnt/tesladisk/TeslaCam/ root@XXX.XXX.XXX.XXX:/mnt/nasXXX/XXX/tesla_videos/
    
    if [ $? -eq 0 ]; then
        mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_STATUS -m "Backup OK"
        mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_TIME -m "$(date +'%H:%M %d/%m')"
    else
        mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_STATUS -m "Error en rsync"
    fi
else
    mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS -t $TOPIC_STATUS -m "Error: Disco no accesible"
fi

# --- PASO 4: Limpieza y Re-conexión del USB ---
sudo umount -l /mnt/tesladisk
sync
# MEJORA 2: Forzar al Tesla a re-detectar el USB (evita la X roja tras sincronizar)
sudo modprobe -r g_mass_storage
sudo modprobe g_mass_storage file=/teslacam.bin stall=0 removable=1


```

## ⏰ Automation (Cron)

Run the script every 15 minutes.

```bash
crontab -e
```

```bash
*/15 * * * * /home/teslacam/sync_tesla.sh >> /home/teslacam/sync.log 2>&1
```

## 🏠 Home Assistant Integration

Add the following sensors to `configuration.yaml`.

```yaml
mqtt:
  sensor:
    - name: "Tesla Pi Status"
      state_topic: "tesla/pi/status"
      icon: "mdi:raspberry-pi"

    - name: "Tesla Pi Last Sync"
      state_topic: "tesla/pi/last_sync"
      icon: "mdi:clock-check"
```

## ⚠️ Important Notes

- **Power:** Use the DATA Micro-USB port (center) for both power and data connection to the Tesla.
- **Filesystem:** If the script reports **Read-Only**, the Tesla is currently writing a clip. The script skips deletion in this state to prevent filesystem corruption.
