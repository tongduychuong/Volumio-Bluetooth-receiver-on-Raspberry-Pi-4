See https://community.volumio.com/t/guide-volumio-bluetooth-receiver/7859


Thanks wolfg1969 https://gist.github.com/wolfg1969/32c3798626ad44ffd4c453114a66ffbe

Set Audio Outout to Hifiberry DAC (my device)

Configure Bluetooth subsystem:

Create file `/etc/bluetooth/audio.conf`
```bash
sudo nano /etc/bluetooth/audio.conf
```
add

```
[General]
Class = 0x200428
Enable = Source,Sink,Media,Socket
```

Set bluealsa-aplay as a service:
Create file `/lib/systemd/system/a2dp-playback.service`

sudo nano /lib/systemd/system/a2dp-playback.service
```
Add
```
[Unit]
Description=A2DP Playback
After=modep-mod-host.service
Requires=modep-mod-host.service

[Service]
Environment=JACK_PROMISCUOUS_SERVER=jack
ExecStartPre=/bin/sleep 3
ExecStart=/usr/bin/bluealsa-aplay
StandardOutput=syslog
StandardError=syslog
SyslogIdentifier=A2DP-Playback
User=volumio

[Install]
WantedBy=multi-user.target
```
```bash
sudo systemctl daemon-reload
```

Add UDEV rules
Create file `/etc/udev/rules.d/99-input.rules`
```bash
sudo nano /etc/udev/rules.d/99-input.rules
```
Add
```
KERNEL=="input[0-9]*", RUN+="/home/volumio/a2dp-autoconnect"
```

Tell Bluetooth how to create a bluetooth->Alsa connection:
Create file `/home/volumio/a2dp-autoconnect`
```bash
nano /home/volumio/a2dp-autoconnect
```
Add

```bash
#!/bin/bash
# Run at each BT connection/disconnection start/stop the service bluealsa-aplay
#!/bin/bash

BTMAC=${NAME//\"/}

function log {
  echo "[$(date)]: $*" >> /var/log/a2dp-autoconnect
}

if echo "$BTMAC" | grep -n -E "^([0-9A-F]{2}:){5}[0-9A-F]{2}$"; then
  if [ "$ACTION" = "remove" ]; then
    log "Stop Played Connection " "$BTMAC"
    sudo systemctl stop bluealsa-aplay@"$BTMAC"
  elif [ "$ACTION" = "add" ]; then
    log "Start Played Connection " "$BTMAC"
    sudo systemctl start bluealsa-aplay@"$BTMAC"
  else
    log "Other action " "$ACTION"
  fi
fi
```

Set the right access permissions:
```bash
sudo chmod a+rwx /home/volumio/a2dp-autoconnect
sudo touch /var/log/a2dp-autoconnect
sudo chmod a+rw /var/log/a2dp-autoconnect
```
REBOOT

After reboot, use bluetoothctl to connect to your bluetooth music source:

```
sudo bluetoothctl
```

```
power on
agent on
default-agent
scan on => xx:xx of your device
pair xx:xx
trust xx:xx
exit
```

On your mobile, connect volumio. Should work.
Once the device is connected you should be able to play something …

Checking

To check if the services are all up and running: 
```bash
systemctl | grep blue
```
You should get something like that:
```
sys-subsystem-bluetooth-devices-hci0.device loaded active plugged /sys/subsystem/bluetooth/devices/hci0

sys-subsystem-bluetooth-devices-hci0:11.device loaded active plugged /sys/subsystem/bluetooth/devices/hci0:11 

bluealsa-aplay@68:FB:7E:24:25:52.service loaded active running BlueAlsa-Aplay 68:FB:7E:24:25:52 -dhw:1,0 

bluealsa.service loaded active running BluezAlsa proxy 

bluetooth.service loaded active running Bluetooth service 

system-bluealsa\x2daplay.slice loaded active active system-bluealsa\x2daplay.slice 

bluetooth.target loaded active active Bluetooth

```

Debug udev rules
```bash
sudo udevadm control --log-priority=debug
sudo journalctl -f
```

