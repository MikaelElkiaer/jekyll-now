---
layout: post
title: XRDP on Ubuntu Mate
---

RDP is a good protocol for remotely accessing a desktop.
I personally use this for gaining remote access to my home network, while at work or elsewhere in the world.

I am running a VM with Ubuntu Mate on a server running Unraid OS.
The following is a guide for how to set up XRDP once you have a Linux machine running somewhere/somehow.
Since I am running Ubuntu Mate the details in the guide will reflect this.

# 1) Install XRDP
This is done using the package manager.

`> sudo apt install xrdp`

# 2) Set up the Mate desktop environment
I initially tried running XFCE and LXDE, on Xubuntu and Lubuntu respectively.
However, I got neither to work, despite spending several hours trying.
However, I got a mate desktop session up and running with Xubuntu and afterwards decided to switch to Ubuntu Mate so that I would not be running different desktops.

`> echo "mate-session" > ~/.xsession`

# 3) Set up TLS
RDP is safe by default - if you are connecting to a machine running a newer version of Windows.
However, XRDP will by default use an unsafe connection.
Luckily, by creating a self-signed certificate and changing 4 lines of config, it is possible to connect using TLS.

## 3a) Create self-signed certificate and key
```
> cd /etc/xrdp
> openssl req -x509 -newkey rsa:2048 -nodes -keyout key.pem -out cert.pem -days 365
```

## 3b) Update xrdp.ini
Update _/etc/xrdp/xrdp.ini_ to
* enable TLS - while disabling older versions of SSL
* point to certificate and key files
```
# /etc/xrdp/xrdp.ini
certificate=/etc/xrdp/cert.pem
key_file=/etc/xrdp/key.pem
security_layer=tls
ssl_protocols=TLSv1.1, TLSv1.2
```

# 4) Open the port
Ubuntu Mate comes with a firewall, so we need to open a port - 3389 is the default port for RDP.

`> ufw allow 3389/tcp`

# 5) Add keyboard layout (optional)
Unfortunately, the built-in Danish layout is wrong.
Luckily, I stumbled upon this guy's GitHub repo where he has a better layout and the updated XRDP keyboard config:

https://github.com/swiatecki/DevNetKeyboard

The commands below, however, are based on my forked version.

```
> cd /etc/xrdp
> wget https://raw.githubusercontent.com/MikaelElkiaer/DevNetKeyboard/master/km-00000406.ini
> chmod 644 km-00000406.ini
> rm xrdp_keyboard.ini
> wget https://raw.githubusercontent.com/MikaelElkiaer/DevNetKeyboard/master/xrdp_keyboard.ini
```

# 6) Restart service
That should be it for setting the service up.
All that is left now is to restart and try it out.

`> service xrdp restart`
