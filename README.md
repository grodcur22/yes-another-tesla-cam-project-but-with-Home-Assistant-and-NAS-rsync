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

# Configuration
MQTT_HOST="<HA_IP>"
MQTT_USER="<USER>"
MQTT_PASS="<PASS>"
TOPIC_STATUS="tesla/pi/status"
TOPIC_TIME="tesla/pi/last_sync"

# 1. Environment Detection
CURRENT_SSID=$(iwgetid -r)
CURRENT_HOUR=$(date +%-H)
CURRENT_MIN=$(date +%-M)

# 2. Travel Logic (Restrict cellular data usage)
if [ "$CURRENT_SSID" == "Tesla" ]; then
    if [ "$CURRENT_HOUR" -ne 3 ] || [ "$CURRENT_MIN" -gt 30 ]; then
        mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
        -t $TOPIC_STATUS -m "Paused (Tesla Network - Off Hours)"
        exit 0
    fi
fi

# 3. Mount & Sync
mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
-t $TOPIC_STATUS -m "Syncing..."

sudo mount -t vfat -o loop,rw,umask=000 /teslacam.bin /mnt/tesladisk

if [ -d "/mnt/tesladisk/TeslaCam" ]; then
    rsync -av /mnt/tesladisk/TeslaCam/ root@<NAS_IP>:/path/to/destination/

    if [ $? -eq 0 ]; then
        if mount | grep /mnt/tesladisk | grep -q "(rw,"; then
            sudo find /mnt/tesladisk/TeslaCam/ -type f -delete 2>/dev/null
            mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
            -t $TOPIC_STATUS -m "Online - Backup OK"
        else
            mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
            -t $TOPIC_STATUS -m "Backup OK (Disk Busy)"
        fi

        mosquitto_pub -h $MQTT_HOST -u $MQTT_USER -P $MQTT_PASS \
        -t $TOPIC_TIME -m "$(date +'%H:%M %d/%m')"
    fi
fi

sudo umount -l /mnt/tesladisk
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
