# RPiVNCHowTo

How to install and configure efficient LAN-based VNC on Raspbian

## Overview

As with practically everything on Linux, there are several different VNC servers to choose from, and there are many ways to install and configure VNC on Raspbian. After a deep dive into VNC, I've identified the most efficient VNC implementation for local LAN use, and have documented it so that you can use and enjoy it as well.

This assumes that you're installing onto a Raspberry Pi on your local LAN running Raspbian, and does not address any extraordinary security concerns such as firewalls, port forwarding, etc. Depending on how your RPi Raspbian security is configured, VNC may log you directly into a desktop, or you may be prompted for credentials.

These instructions have been tested on 2018-11-13-raspbian-stretch, and details for both Full and Lite are included.

There are (at least) 3 different VNC servers provided with Raspbian. RealVNC provides a cloud-based solution, and is quite elegant. However, it requires that you have an account with the RealVNC service in the cloud.

TightVNC is a very nice VNC server, as well, but I found that certain fonts were corrupted (9x15bold, specifically, which I have used for far too long in my xterm windows).

The third VNC server is TigerVNC, which is extremely flexible. The rest of this document is based on TigerVNC. (As an aside, tightvnc can be used with this method as well, however, the systemd service commands in the .service files are specific to TigerVNC and will need adjustment for tightvnc by changing Xtigervnc to Xvnc, in addition to installing tightvncserver.)

TigerVNC provides a tool that greatly simplifies running a VNC server (/usr/bin/vncserver). However, it isn't optimal in that it keeps the VNC server running whether you're using it or not, and you need to either manually start it or sort out how to automaticallly start it when the system reboots. It is, however, a much simpler way to get started. The method described here starts the VNC server on-demand, and is fully integrated into Linux systemd.

*Should you use /usr/bin/vncserver or this method?* If you have never typed into a terminal on Linux, vncserver was made for you. On the other hand, if you're moderately comfortable working with the Linux bash command line and making simple file edits, the method documented here is a much better approach. 

## Installing onto Raspbian Full

Full Raspbian comes with the realvnc-vnc-server package installed. You can remove it or not. Then execute the following commands:

* `sudo apt-get install tigervnc-standalone-server tigervnc-common package xfonts-scalable xfonts-100dpi xfonts-75dpi`

Now continue with the System Configuration below.

## Installing onto Raspbian Lite

Lite Raspbian is exactly that...Lite. You'll need to install a few more packages. 

* `apt-get install xserver-xorg xserver-xorg-core xserver-common xterm xfonts-base`
* `apt-get install tigervnc-standalone-server tigervnc-common package xfonts-100dpi xfonts-75dpi xfonts-scalable`
* You'll need a *Display Manager*. I prefer xdm, but lightdm is another good choice. `apt-get install xdm` or 'apt-get install lightdm` as appropriate.
* You'll also need a *Window Manager*. I prefer icewm, but you might prefer something different. In any case, you'll need to `apt-get install` your Window Manager.

## System configuration

This section details the system configuration changes to enable VNC.

### Display Manager configuration

The most popular Display Manager is lightdm. I prefer xdm, so I've documented that as well. In either case, the Display Manager must have XDMCP enabled, so that VNC can create the desktop.

#### Lightdm configuration

Edit /etc/lightdm/lightdm.conf and modify the XDMCPServer section as follows

    [XDMCPServer]
    enabled=true
    port=177

#### xdm configuration

* Edit /etc/X11/xdm-config and comment out the line *DisplayManager.requestPort* with a "!"
* Edit /etc/X11/Xaccess and uncomment the line that has *#any host can get a login window*
* `sudo systemctl try-restart xdm.service`

### VNC Service configuration

* Create or download and update the systemd configuration files as appropriate. Each VNC service implements a single screen resolution.
    * Download (or create) the files
        * You can download the files or copy/paste them from the bottom of this document
        * To download:
            * Bash> cd /lib/systemd/system
            * Bash> sudo wget https://raw.githubusercontent.com/gitbls/RPiVNCHowTo/master/xvnc0.socket
            * Bash> sudo wget https://raw.githubusercontent.com/gitbls/RPiVNCHowTo/master/xvnc0\@.service
    * Edit the VNC configuration files
        * Use sudo with your favorite editor to edit the .service files, which are in /lib/systemd/system
        * Change the resolution in xvnc0@.service as desired. I like my VNC window to be nearly full screen size on my 1900x1200 monitor, so I use 1880x1100, which is the setting in xvnc0@.service. For my 1900x1080 laptop I use 1880x960, which is in xvnc1@.service.
        * The filenames for the .socket and the .service file must match, except for the @ in the .service filename. You can name them fred.socket and fred@.service if you'd prefer.
        * The @ in the filename is important. When a VNC connection is made, a new service is started with the name similar to xvnc0@n-serveripaddr:port-remoteipaddr:port.service. the @ enables that. You can view the status of the services with `sudo systemctl status xvnc*`
* sudo systemctl daemon-reload
* Start the VNC service connections. For each one you've created (e.g., xvnc0 and xvnc1 in this example), start the socket and enable it across reboots
    * sudo systemctl start xvnc0.socket
    * sudo systemctl enable xvnc0.socket
* Test that you can use a VNC client to connect to your server

If additional VNC resolutions are needed, copy the xvnc0* files to, for instance, xvnc1.socket and xvnc1@.socket. Then edit the .socket file to change the socket number, and edit the .service file to change the resolution. After the files are appropriately updated, issue the above systemctl start/enable commands for the new socket.

## How does this work?

Once you start the xvnc0.socket, the systemd process listens on the specified TCP port. When a network connect lands on that port, systemd starts the service described by the .service with the filename corresponding to the socket name. This is quite simple and elegant, and in general, *just works*.

In the event that it doesn't, see the next section.

## Problem solving

If things aren't working correctly, the best approach is to look at the system log (with `sudo journalctl`) and also the status of the services with `sudo systemctl status xvnc*`

If a VNC window pops up but there's no graphical display or login box, you're probably using xdm and have run into a problem that I encountered. I'm not sure what the root cause is, but here are two fixes. Either one is fine, but changing the service configuration file has the least likelihood of breaking something else.

* Change -query localhost to -query 127.0.0.1 in the xvnc@ files, then do sudo systemctl daemon-reload
* Edit /etc/hosts and on the line starting with ::1 change *localhost *to *localhost6*

## systemd configuration file listings

### xvnc0.socket

```
    [Unit]
    Description=XVNC Server
    
    [Socket]
    ListenStream=5900
    Accept=yes
    
    [Install]
    WantedBy=sockets.target
```

### xvnc0@.service

```
    [Unit]
    Description=XVNC Per-Connection Daemon
    
    [Service]
    ExecStart=-/usr/bin/Xtigervnc -noreset -inetd -query localhost -geometry 1880x1100 -pn -once -SecurityTypes None
    User=nobody
    StandardInput=socket
    StandardOutput=socket
    StandardError=syslog
```





