See https://community.volumio.com/t/guide-volumio-bluetooth-receiver/7859
Thanks wolfg1969 https://gist.github.com/wolfg1969/32c3798626ad44ffd4c453114a66ffbe

Install dependencies:
```bash
sudo apt-get update 
sudo apt-get install dh-autoreconf libasound2-dev libortp-dev pi-bluetooth
sudo apt-get install libusb-dev libglib2.0-dev libudev-dev libical-dev libreadline-dev libsbc1 libsbc-dev
```

Compile Bluez 5.51:
```bash
git clone git://git.kernel.org/pub/scm/bluetooth/bluez.git
cd bluez
git checkout 5.51
./bootstrap
./configure --enable-library --enable-experimental --enable-tools
make
sudo make install

sudo ln -s /usr/local/lib/libbluetooth.so.3.19.0 /usr/lib/arm-linux-gnueabihf/libbluetooth.so
sudo ln -s /usr/local/lib/libbluetooth.so.3.19.0 /usr/lib/arm-linux-gnueabihf/libbluetooth.so.3
sudo ln -s /usr/local/lib/libbluetooth.so.3.19.0 /usr/lib/arm-linux-gnueabihf/libbluetooth.so.3.19.0
```
Compile Bluez-Alsa
```bash
cd
git clone https://github.com/Arkq/bluez-alsa.git
cd bluez-alsa
git checkout v4.0.0
autoreconf --install
mkdir build && cd build
../configure --disable-hcitop --with-alsaplugindir=/usr/lib/arm-linux-gnueabihf/alsa-lib 
make
sudo make install
```

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

0x200428 - Hifi Audio Device, see http://bluetooth-pentest.narod.ru/software/bluetooth_class_of_device-service_generator.html

Update file `/etc/bluetooth/main.conf`

```bash
sudo nano /etc/bluetooth/main.conf
```
add

```
[General]
Class = 0x200428
```

Automate BluezAlsa:
Set BlueAlsa as a service
Create file `/lib/systemd/system/bluealsa.service`
```bash
sudo nano /lib/systemd/system/bluealsa.service
```
Add
```
[Unit]
Description=BluezAlsa proxy
Requires=bluetooth.service
After=bluetooth.service

[Service]
Type=simple
User=root
Group=audio
ExecStart=/usr/bin/bluealsa -p a2dp-source -p a2dp-sink

[Install]
WantedBy=multi-user.target
```

Enable BluezAlsa starts from boot:
```bash
sudo systemctl daemon-reload && sudo systemctl enable bluealsa.service
```
Set bluealsa-aplay as a service:
Create file `/lib/systemd/system/bluealsa-aplay@.service`
hw2:0 is the hifiberry audio device I want to use. There may be better way to link it. Use `aplay -l` or `aplay -L` to find the device (https://superuser.com/questions/53957/what-do-alsa-devices-like-hw0-0-mean-how-do-i-figure-out-which-to-use).
```bash
sudo nano /lib/systemd/system/bluealsa-aplay@.service
```
Add
```
[Unit] 
Description=BlueAlsa-Aplay %I -Dhw:2,0
Requires=bluetooth.service bluealsa.service

[Service]
Type=simple
User=volumio
Group=audio
ExecStart=/usr/bin/bluealsa-aplay %I -Dhw:2,0
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

