# RPiVNCHowTo

How to install and configure efficient LAN-based VNC on Raspbian

## Overview

There are several different VNC servers to choose from on Linux and Raspbian, and there are many ways to install and configure VNC. After an unexpected deep dive into VNC, I've identified the most efficient VNC implementation for local LAN use, and have documented it so that you can use and enjoy it as well.

This guide assumes that you're installing onto a Raspberry Pi on your local LAN running Raspbian, and does not address any extraordinary security concerns such as firewalls, port forwarding, etc. Depending on how your RPi Raspbian security is configured, VNC may log you directly into a desktop, or you may be prompted for credentials.

These instructions have been tested on 2018-11-13-raspbian-stretch, and details for both Full and Lite are included.

**NOTE: **This method can be used on other Linux distros with a recent systemd implementations, although adjustments will be required.

There are (at least) 3 different VNC servers provided with Raspbian. **RealVNC** provides a cloud-based solution, and is quite elegant. However, it may require that you have an account with the RealVNC service in the cloud. This may be a bit over-the-top for a LAN-based configuration.

**TightVNC** is a very nice VNC server, as well, but I found that at least one font was not always displayed correctly (9x15bold, specifically, which I have used for far too long in my xterm windows).

The third VNC server is **TigerVNC**. The rest of this document is based on TigerVNC. (As an aside, TightVNC can be used with this method as well, however, the ExecStart command in the .service files that starts VNC is specific to TigerVNC and will need adjustment for tightvnc by changing `Xtigervnc` to `Xtightvnc`, in addition to installing the tightvncserver package.)

TigerVNC provides a tool that greatly simplifies running a VNC server (`/usr/bin/vncserver`). However, it isn't optimal in that it keeps the VNC server running whether you're using it or not, and you need to either manually start it or sort out how to automaticallly start it when the system reboots. It is, however, a much simpler way to get started. The method      described here starts the VNC server on-demand, and is fully integrated into Linux systemd.

*Should you use /usr/bin/vncserver or this method?* If you have never typed into a terminal on Linux, `vncserver` was made for you. On the other hand, if you're moderately comfortable working with the Linux bash command line and are able to make simple file edits, the method documented here is a much better approach. 

## Installing onto Raspbian Full

Full Raspbian comes with the realvnc-vnc-server package installed. You can remove it or not. Then execute the following commands:

* `sudo apt-get install tigervnc-standalone-server tigervnc-common xfonts-scalable xfonts-100dpi xfonts-75dpi`

Continue with the System Configuration below.

## Installing onto Raspbian Lite

Lite Raspbian is exactly that...Lite. You'll need to install a few more packages. 

* `sudo apt-get install xserver-xorg xserver-xorg-core xserver-common xterm xfonts-base`
* `sudo apt-get install tigervnc-standalone-server tigervnc-common package xfonts-100dpi xfonts-75dpi xfonts-scalable`
* You'll need a *Display Manager*. I prefer xdm, but lightdm is another good choice. `sudo apt-get install xdm` or `sudo apt-get install lightdm` as appropriate.
* You'll also need a *Window Manager*. I prefer icewm, but you might prefer something different. In any case, you'll need to `sudo apt-get install` your Window Manager of choice.

## System Configuration

This section details the system configuration changes to enable VNC.

### Display Manager Configuration

The most popular Display Manager is lightdm, and lightdm is installed by default on Raspbian Full. I've documented xdm as well. In either case, the Display Manager must have XDMCP enabled, so that VNC can create the desktop.

#### Lightdm Configuration

Edit `/etc/lightdm/lightdm.conf` and modify the XDMCPServer section as follows

```
[XDMCPServer]
enabled=true
port=177
```

`sudo systemctl try-restart lightdm.service` to restart the service with the new settings.

#### xdm Configuration

xdm is a lightweight Display Manager with less capabilities than lightdm. That said, I've found it to consume less system resources than lightdm, and coupled with choosewm (see Appendix below), to meet my needs.

* Edit `/etc/X11/xdm/xdm-config` and comment out the line *DisplayManager.requestPort* with a "!"
* Edit `/etc/X11/xdm/Xaccess` and uncomment the line that has *#any host can get a login window*
* `sudo systemctl try-restart xdm.service` to restart the service with the new settings

**Note**If your system has lightdm installed and you want to switch to xdm (or vice versa), the package installer will ask you which one you want to use. After completing the installation, reboot the system for the change to take effect. To switch between the two display managers once they are both installed, use `dpkg-reconfigure lightdm`.

### VNC Service configuration

* Create or download and update the systemd configuration files as appropriate. Each VNC service, consisting of a .socket and .service file, implements a single screen resolution.
    * Download (or create) the files
        * You can download the files or copy/paste them from the bottom of this document
        * To download:
            * Bash> cd /lib/systemd/system
            * Bash> sudo wget ht<span>tps://</span>raw.githubusercontent.com/gitbls/RPiVNCHowTo/master/xvnc0.socket
            * Bash> sudo wget ht<span>tps://</span>raw.githubusercontent.com/gitbls/RPiVNCHowTo/master/xvnc0%40.service
    * Edit the VNC configuration files
        * Use sudo with your favorite editor to edit the .service files, which are in /lib/systemd/system
        * Change the resolution in xvnc<span>0@</span>.service as desired. I like my VNC window to be nearly full screen size on my 1900x1200 monitor, so I use 1880x1100, which is the setting in xvnc0@.service. For my 1900x1080 laptop I use 1880x960, which I've put in xvnc<span>1@.</span>service.
        * The filenames for the .socket and the .service file must match, except for the @ in the .service filename. (Yes, you can name them fred.socket and fred<span>@.</span>service if you'd prefer, although this is not recommended.)
        * The @ in the filename is important. When a VNC connection is made, a new service is started with the name similar to xvnc0@n-serveripaddr:port-remoteipaddr:port.service. the @ enables that.
        *  **Note** that you may need to prefix the "@" with a backslash ("\") on the command line.
* `sudo systemctl daemon-reload` - Must be done any time the systemd configuration files are modified.
* Start the VNC service connections. For each one you've created (e.g., xvnc0 and xvnc1 in this example), start the socket and enable it across reboots
    * `sudo systemctl start xvnc0.socket` - Starts the socket in the running system
    * `sudo systemctl enable xvnc0.socket` - Sets the socket to start after the system reboots
* **Test** that you can use a VNC client to connect to your server. 

If additional VNC resolutions are needed, copy the xvnc0* files to, for instance, xvnc1.socket and xvnc1@.socket. Then edit the .socket file to change the socket number (increase by 1 from the previous), and edit the .service file to change the resolution. After the files are appropriately updated, issue the above systemctl daemon-reload/start/enable commands for the new socket.

## How does this work?

Once you start the xvnc0.socket, the systemd process listens on the specified TCP port. When a network connect lands on that port, systemd starts the service described by the .service file with the filename corresponding to the socket name. This is quite simple and elegant, and in general, *just works*.

In the event that it doesn't, see the next section.

## Problem Solving

If things aren't working correctly, the best approach is to look at the system log (with `sudo journalctl`) and also the status of the services with `sudo systemctl status xvnc*`

If a VNC window pops up but there's no graphical display or login box, you're probably using xdm and have run into a problem that I encountered. I'm not sure yet what the root cause is, but it's easy to fix:

* Change `-query localhost` to `-query 127.0.0.1` in the xvnc<span>@.</span>service files, then do `sudo systemctl daemon-reload` and try again.

## systemd configuration file listings

### xvnc0.socket

The .socket file describes the socket to systemd. VNC socket numbers start at 5900. It is suggested that you maintain these in sequential order, so that a) the stream number can be inferred from the filename (e.g., xvnc0=5900, xvnc1=5901, etc), and your future sanity is not jeapordized.

```
[Unit]
Description=XVNC Server

[Socket]
ListenStream=5900
Accept=yes

[Install]
WantedBy=sockets.target
```

### xvnc<span>0@.</span>service

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
[ -f "$xchoose" ] && exec $xchoose
[ -f "$wm" ] && exec $wm
#
# If we get here, something is wrong, so run an xterm
#
exec /usr/bin/xterm
```
