# RPiVNCHowTo

How to install and configure efficient LAN-based VNC for virtual desktops on Raspberry Pi OS (RasPiOS).

## Overview

There are several different VNC servers to choose from on Linux and RasPiOS, and there are many ways to install and configure VNC. This is one of the most efficient and lightweight VNC implementations for local LAN use. I've documented it so that you can use and enjoy it as well.

Once VNC is set up according to this guide, each user can use their favorite VNC client to connect, and will log in using their own username/password, receiving their own virtual desktop. Each user can customize their virtual desktop as they desire.

This guide assumes that you're installing onto a Raspberry Pi on your local LAN running RasPiOS, and does not address any extraordinary security concerns such as firewalls, port forwarding, etc. Depending on how your RPi RasPiOS security is configured, VNC may log you directly into a desktop, or you may be prompted for credentials. 

These instructions have been tested on RasPiOS Buster as well as Raspbian Stretch, and details for both Full and Lite are included. In the following notes, read "sudo Edit" as "sudo nano/vi/emacs/..." as appropriate for the editor you use. 

If you find this github useful, please consider starring it to help me understand how many people are using it. Thanks!

NOTE: This can be used on other Linux distros with a recent systemd implementation, although minor adjustments may be required. If you run into problems, please let me know!

There are (at least) 3 different VNC servers provided with RasPiOS. **RealVNC** is quite elegant, comes pre-installed on RaspiOS
Full Desktop, and can be used on a LAN without a RealVNC Cloud account. Using a RealVNC Cloud account provides additional
functionality. RealVNC for personal use is Free, and if you use a Cloud account there is a limit of 5 internet-connected computers.

But, Linux is all about choice, and there are at least 2 other VNC servers available on Linux. The rest of this document is about how to most effectively configure these other VNC servers.

This method only configures VNC *virtual desktops*. If you want to connect to the system console, you probably want to do it for the best performance. In that case, you should probably use RealVNC, which is much more optimized for that use case. And note that you can use these virtual desktops alongside RealVNC Server, and have the best of both worlds. **NOTE:** Only TigerVNC cooperatively coexists with RealVNC. Installing TightVNC server seems to cause RealVNC Server to be uninstalled.

**TightVNC** is a very nice VNC server, as well, but I found that at least one font was not always displayed correctly (9x15bold, specifically, which I have used for far too long in my xterm windows). Additionally, installing TightVNC forces the removal of realvnc-vnc-server, which is needed if you want to access the Console session in addition to virtual sessions.

The third VNC server is **TigerVNC**. The rest of this document is based on TigerVNC. (As an aside, TightVNC can be used with this method as well, however, the ExecStart command in the .service files that starts VNC is specific to TigerVNC and will need adjustment for tightvnc by changing `Xtigervnc` to `Xtightvnc`, in addition to installing the tightvncserver package.) **NOTE:**`make-systemd-xvnc` adjusts the command lines automatically for TightVNC.

TigerVNC (and TightVNC and RealVNC as well) provide a tool that greatly simplifies running a VNC server (`/usr/bin/vncserver`) with little to no config file editing. However, this method isn't optimal from a system resources perspective in that it keeps a VNC server running whether you're using it or not, and you need to either manually start it or sort out how to automaticallly start it when the system reboots. It is, however, a much simpler way to get started for some people. The method described here starts the requested VNC server on-demand, and is fully integrated into Linux systemd.

*Should you use /usr/bin/vncserver or this method?* If you have never typed into a terminal on Linux, `vncserver` was made for you. On the other hand, if you're moderately comfortable working with the Linux bash command line and are able to make simple file edits, the method documented here is a much efficient.

## A Note about RealVNC and these virtual desktops

It is absolutely possible to use RealVNC server for graphical console access and virtual desktops enabled by this github on the same system. In fact, if you'd like to do this as easily as possible, I encourage to take a close look at [sdm](https://github.com/gitbls/sdm).

sdm makes it super-easy to configure RasPiOS (Desktop or Lite) with all your favorite settings, apps and configurations (my fairly fully-loaded IMG takes about 10 minutes to configure on a Pi4). Then, when you need a new SD Card, you just burn one from your customized image. The system comes up with all Localization settings in place, WiFi configured, apps installed, etc. And, one of the things that sdm will configure for you is VNC. Have a look!

But, if you'd prefer to not use sdm, that's fine. RealVNC Server (which attaches to the Pi4 graphical console) works great with TigerVNC-powered virtual desktops installed using the technique on this github. Prior to Feb 20 2021 make-systemd-xvnc assigned virtual desktops to ports starting at 5900. Unfortunately, RealVNC Server uses port 5900. So, make-systemd-xvnc and this documentation now skips 5900 for compatibility.

So, either using sdm or this technique, if you want to have RealVNC and TigerVNC-powered Virtual Desktops on the same system, it just works.

## Installing onto RasPiOS Full

Full RasPiOS comes with the realvnc-vnc-server package installed. You can remove it or not. Then execute the following commands:

* **TigerVNC server:** `sudo apt-get install tigervnc-standalone-server xfonts-scalable xfonts-100dpi xfonts-75dpi`
* **TightVNC server:** `sudo apt-get install tightvncserver xfonts-scalable xfonts-100dpi xfonts-75dpi`

Continue by performing the following steps, detailed under System Configuration below:

* Edit /etc/lightdm/lightdm.conf and enable XDMCPServer (manually or use edit-lightdm-config on this github)
* Create the xvnc service files as documented in VNC Service Configuration
* sudo systemctl enable xvnc1.socket, and any other sockets that you've created
* Reboot and you're done!
* **DO NOT** install any of the packages suggested below for RasPiOS Lite

NOTE: If you are going to have multiple different users login to the same Pi, you'll need to disable AutoLogin in `sudo raspi-config`: System Options, S5(Boot/Auto Login), and then set B3 (Desktop).

## Installing onto RasPiOS Lite

RasPiOS Lite is exactly that...Lite. You'll need to install a few more packages. 

* **Install** the basic XServer software and xterm
    * `sudo apt-get install xserver-xorg xserver-xorg-core xserver-common xterm xfonts-base xfonts-100dpi xfonts-75dpi xfonts-scalable`
* **TigerVNC Server:** `sudo apt-get install tigervnc-standalone-server`
* **TightVNC server:** `sudo apt-get install tightvncserver`
* You'll need a *Display Manager*. I prefer xdm, but wdm and lightdm are good choices as well. `sudo apt-get install xdm` or `sudo apt-get install lightdm` as appropriate.
* You'll also need a *Window Manager*. I prefer icewm, but you might prefer something different. In any case, you'll need to `sudo apt-get install` your Window Manager of choice.

## System Configuration

This section details the system configuration changes to enable virtual VNC desktops.

### Display Manager Configuration

The most popular Display Manager on RasPiOS is lightdm, and lightdm is installed by default on RasPiOS Full. I've documented xdm as well. In either case, the Display Manager must have XDMCP enabled, so that VNC can create the desktop.

In either case, on RasPiOS Lite, after updating the display manager configuration per the following details, issue this command on the console: `sudo systemctl set-default graphical.target`.

#### Lightdm Configuration

sudo Edit `/etc/lightdm/lightdm.conf` and modify the XDMCPServer section as follows

```
[XDMCPServer]
enabled=true
port=177
```

The `edit-lightdm-config` script on this github can be used to make this edit.

After you've completed this change, `sudo systemctl restart lightdm.service` to restart the service with the new settings (this will log out your session!), or restart your system. If you don't, you'll only see a black screen when you connect to the system with VNC.

If you prefer to not have lightdm to run on the (HDMI) console, that is, you only want a command-line console, sudo edit /etc/lightdm/lightdm.conf again and after the line `#start-default-seat=true` add this new line.
```
start-default-seat=false
```

#### xdm Configuration

xdm is a lightweight Display Manager with less capabilities than lightdm. That said, I've found it to consume less system resources than lightdm, and meets my needs when coupled with choosewm (see Appendix below).

* sudo Edit `/etc/X11/xdm/xdm-config` and comment out the line *DisplayManager.requestPort* by adding a "!" at the start of the line.`
* sudo Edit `/etc/X11/xdm/Xaccess` and uncomment the line that has *#any host can get a login window* by removing the '#' at the start of the line.
* If you don't want a graphical login on the Raspberry Pi HDMI port sudo Edit `/etc/X11/xdm/Xservers` and comment out the line `:0 local /usr/bin/X :0 vt7 -nolisten tcp` by adding a '#' at the start of the line.
* `sudo systemctl restart xdm.service` to restart the service with the new settings

The edit-xdm-config script (on this github) can be used to make the above modifications.

**Note**: If your system has both lightdm and xdm installed and you want to switch to xdm (or vice versa), the package installer will ask you which one you want to use. After completing the installation, reboot the system for the change to take effect. To switch between the two display managers if they are both installed, use `dpkg-reconfigure lightdm`.

### VNC Service configuration

**NOTE** If you aren't comfortable editing system configuration files, I strongly encourage you to use the make-systemd-xvnc script on this github. The following explanation provides background on how to edit the configuration files, but these are not exhaustive and step-by-step.

* A systemd socket/service config file pair is required for each VNC port, and each port will implement a single screen resolution. You can easily create the socket/service pair files with the make-systemd-xvnc script on this github or you can create your own in /etc/systemd/system (but make-systemd-xvnc is MUCH easier).
    * Edit the VNC configuration files as needed. If you used make-systemd-xvnc you should not need to modify them unless you run into a problem.
        * If you need to edit them, sudo Edit the .service files as appropriate, located in /etc/systemd/system
        * Change the resolution in xvnc<span>1@</span>.service as desired. I like my VNC window to be nearly full screen size on my 1900x1200 monitor, so I use 1880x1100, which is the setting in xvnc<span>1@.</span>service. For my 1900x1080 laptop I use 1880x960, which I've put in the file xvnc<span>2@.</span>service (with a corresponding xvnc2.socket file).
        * The filenames for the .socket and the .service file must match, except for the @ in the .service filename.
        * The @ in the filename is important. When a VNC connection is made, a new service is automatically started with the name similar to xvnc1@n-serveripaddr:port-remoteipaddr:port.service. the @ enables that.
* `sudo systemctl daemon-reload` - Must be done when systemd configuration files are modified. If you use make-systemd-xvnc you are given the option of performing the reload and starting the sockets.
* Start the VNC service connections. For each one you've created (e.g., xvnc1 and xvnc2 in this example), start the socket and enable it across reboots:
    * `sudo systemctl start xvnc1.socket` - Starts the socket in the running system
    * `sudo systemctl enable xvnc1.socket` - Sets the socket to start after the system reboots
    * `sudo systemctl enable --now xvnc1.socket` - Starts the socket in the running system AND sets the socket to start after system reboots
* **Test** that you can use a VNC client to connect to your server. If you get a black screen in the VNC window make sure that you have restarted lightdm, or restart the system.

If you are using make-systemd-xvnc you can easily create all the socket/service pairs. When you create them manually, if additional VNC resolutions are needed, copy the xvnc1* files to, for instance, xvnc2.socket and xvnc<span>2@.</span>service. Then, sudo Edit the .socket file to change the socket number (increase by 1 from the previous), and sudo Edit the .service file to change the resolution. After the files are appropriately updated, issue the above sudo systemctl daemon-reload/start/enable command sequence for the new socket.

`make-systemd-xvnc` looks for existing socket/service pair files, and if found will ask if you want to replace or create additional pairs. Newly-create pairs are numbered following the last xvnc socket/service pair file.

## What about using this remotely? Is it secure?

The VNC protocol by itself is insecure, so you shouldn't use it over insecure networks, such as the internet, without taking precautions. With the solution documented here you need to use a VPN or similar solution (ssh, meet-in-the middle, etc) to ensure a secure connection. As an aside, I assume that RealVNC's Cloud Connections use an enhanced VNC protocol to ensure high security.

## How does this work?

Once you start the xvnc1.socket, the systemd process listens on the specified TCP port. When a network connect lands on that port, systemd starts the service described by the .service file with the filename corresponding to the socket name. This is quite simple and elegant, and it *just works*.

In the event that it doesn't, see the next section.

## Problem Solving

If things aren't working correctly, the best approach is to look at the system log (with `sudo journalctl -b`) and also the status of the services with `sudo systemctl status xvnc*`

## systemd configuration file listings

### xvnc1.socket

The .socket file describes the socket to systemd. VNC socket numbers start at 5900, but avoid using 5900 in case you ever want to use RealVNC Server. It is suggested that you maintain these in sequential order, so that a) the stream number can be inferred from the filename (e.g., xvnc1=5901, xvnc2=5902, etc), and your future sanity is not jeapordized.

```
[Unit]
Description=XVNC Server

[Socket]
ListenStream=5901
Accept=yes

[Install]
WantedBy=sockets.target
```

### xvnc<span>1@.</span>service

```
[Unit]
Description=XVNC Per-Connection Daemon

[Service]
ExecStart=-/usr/bin/Xtigervnc -noreset -inetd -query 127.0.0.1 -geometry 1880x1100 -pn -once -SecurityTypes None
StandardInput=socket
StandardOutput=socket
StandardError=journal
```
## Appendix 1 - xdm and selecting a Window Manager

xdm does not provide a built-in way (that I'm aware of) to select the Window Manager on login. The package choosewm fills the gap. Simply `apt-get install choosewm` to install it. Choosewm uses the system's list of Window Managers to offer you a choice when you login.

You'll need a ~/.xsession file to enable choosewm. Here's a sample:

```
# set wm to default window manager
# set xchoose to "/usr/bin/choosewm" or "" to not run choosewm and use default
wm="/usr/bin/icewm"
xchoose=""
# Comment out next line to not run choosewm
xchoose="/usr/bin/choosewm"
xrdb < ~/.Xdefaults
xhost +
#
# Personal customizations
#
xsetroot -solid black
#
# run choosewm (which will run the selected wm) or the default wm
#
[ -x "$xchoose" ] && exec $xchoose
[ -x "$wm" ] && exec $wm
#
# If we get here, something is wrong, so run an xterm
#
exec /usr/bin/xterm
```
