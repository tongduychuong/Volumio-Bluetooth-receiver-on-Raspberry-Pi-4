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

```bash
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

