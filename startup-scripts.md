* Startup Scripts on Raspberry Pi

** Python Scripts

SystemdForUpstartUsers - Ubuntu Wiki
https://wiki.ubuntu.com/SystemdForUpstartUsers
 
Make unit file, modify WorkingDir and ExecStart:

```bash
root@picam4:/# cat /etc/systemd/system/camera.service
[Unit]
Description=DigitalTwin Camera Service
After=network.target
StartLimitIntervalSec=0
 
[Service]
Type=simple
Restart=always
RestartSec=1
User=pi
WorkingDir=/home/pi/Dev/ZmqSender.New
ExecStart=/home/pi/Dev/ZmqSender.New/.env/bin/python /home/pi/Dev/ZmqSender.New/sender-secure.py
 
[Install]
WantedBy=multi-user.target
```

Enable, so it starts on reboot

```bash
root@picam4:/# systemctl enable camera
Created symlink /etc/systemd/system/multi-user.target.wants/camera.service → /etc/systemd/system/camera.service.
```

Can also stop/start/status etc.


```bash
[Service]
…
Environment="AWS_ACCESS_KEY_ID=[--access--key--]"
Environment="AWS_SECRET_ACCESS_KEY=[--secret-access-key--]"
Environment="AWS_DEFAULT_REGION=us-west-2"
…
ExecStart=/home/pi/Dev/ZmqSender.New/.env/bin/python /home/pi/Dev/ZmqSender.New/sender-secure.py
```

** NodeJs Scripts

PM2 - Home (keymetrics.io) recently, for some Node services, which worked well. Can also be used for non-node processes.
 
