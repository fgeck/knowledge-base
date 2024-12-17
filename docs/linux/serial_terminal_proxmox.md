# Setting Up a Serial Terminal in Proxmox

Reference for setting up a serial terminal in proxmox can be found [here](https://pve.proxmox.com/wiki/Serial_Terminal).
Unfortunately this does not cover the process for debian as there is not Upstart init system.

To use it with systemd do the following:

## Proxmox HOST

```bash
qm set <VM_ID> -serial0 socket
```

## Proxmox GUEST VM

* edit grub

```bash
sudo nano /etc/defaults/grub
```

* Paste

```init
GRUB_CMDLINE_LINUX_DEFAULT="quiet console=tty0 console=ttyS0,115200n8"
GRUB_TERMINAL=serial
GRUB_SERIAL_COMMAND="serial --speed=115200 --unit=0 --word=8 --parity=no --stop=1"
```

* update grub

```bash
sudo update-grub
```

* Create the service


```bash
sudo nano /etc/systemd/system/serial-getty@ttyS0.service
```

* Paste following content:

```ini
[Unit]
Description=Serial Getty on ttyS0
Documentation=man:agetty(8)
After=systemd-user-sessions.service plymouth-quit-wait.service
After=rc-local.service

[Service]
ExecStart=/sbin/agetty -L 115200 ttyS0 vt102
Restart=always
RestartSec=0
UtmpIdentifier=ttyS0
TTYPath=/dev/ttyS0
TTYReset=yes
TTYVHangup=yes
KillMode=process
IgnoreSIGPIPE=no
SendSIGHUP=yes

[Install]
WantedBy=multi-user.target
```

* Reload Systemd,Enable and Start the Service:

```bash
sudo systemctl daemon-reload
sudo systemctl enable serial-getty@ttyS0.service
sudo systemctl start serial-getty@ttyS0.service
```

* Verify:

```bash
sudo systemctl status serial-getty@ttyS0.service
```

Now you should be able to use xterm.js in proxmox GUI
