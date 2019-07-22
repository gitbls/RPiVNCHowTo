# RPiVNCHowTo

How to install and configure efficient LAN-based VNC on Raspbian

## Overview

There are several different VNC servers to choose from on Linux and Raspbian, and there are many ways to install and configure VNC. This is one of the most efficient and lightweight VNC implementations for local LAN use. I've documented it so that you can use and enjoy it as well.

Once VNC is set up according to this guide, each user will use their favorite VNC client to connect, and will log in using their own username/password, receiving their own virtual desktop. Each user can customize their virtual desktop as they desire.

This guide assumes that you're installing onto a Raspberry Pi on your local LAN running Raspbian, and does not address any extraordinary security concerns such as firewalls, port forwarding, etc. Depending on how your RPi Raspbian security is configured, VNC may log you directly into a desktop, or you may be prompted for credentials.

These instructions have been tested on 2019-06-20 Raspbian Buster as well as Stretch, and details for both Full and Lite are included. In the following notes, read "sudo Edit" as "sudo nano/vi/emacs/..." as appropriate for the editor you use.

**NOTE: **This method can be used on other Linux distros with a recent systemd implementations, although minor adjustments will be required.

There are (at least) 3 different VNC servers provided with Raspbian. **RealVNC** provides a cloud-based solution, and is quite elegant. However, it may require that you have an account with the RealVNC service in the cloud. This may be a bit over-the-top for a LAN-based configuration.

**TightVNC** is a very nice VNC server, as well, but I found that at least one font was not always displayed correctly (9x15bold, specifically, which I have used for far too long in my xterm windows).

The third VNC server is **TigerVNC**. The rest of this document is based on TigerVNC. (As an aside, TightVNC can be used with this method as well, however, the ExecStart command in the .service files that starts VNC is specific to TigerVNC and will need adjustment for tightvnc by changing `Xtigervnc` to `Xtightvnc`, in addition to installing the tightvncserver package.)

TigerVNC (and TightVNC and RealVNC as well) provide a tool that greatly simplifies running a VNC server (`/usr/bin/vncserver`) with little to no config file editing. However, this method isn't optimal from a system resources perspective in that it keeps a VNC server running whether you're using it or not, and you need to either manually start it or sort out how to automaticallly start it when the system reboots. It is, however, a much simpler way to get started. The method      described here starts the VNC server on-demand, and is fully integrated into Linux systemd.

*Should you use /usr/bin/vncserver or this method?* If you have never typed into a terminal on Linux, `vncserver` was made for you. On the other hand, if you're moderately comfortable working with the Linux bash command line and are able to make simple file edits, the method documented here is a much efficient.

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

sudo Edit `/etc/lightdm/lightdm.conf` and modify the XDMCPServer section as follows

```
[XDMCPServer]
enabled=true
port=177
```

`sudo systemctl restart lightdm.service` to restart the service with the new settings.

#### xdm Configuration

xdm is a lightweight Display Manager with less capabilities than lightdm. That said, I've found it to consume less system resources than lightdm, and coupled with choosewm (see Appendix below), to meet my needs.

* sudo Edit `/etc/X11/xdm/xdm-config` and comment out the line *DisplayManager.requestPort* with a "!"
* sudo Edit `/etc/X11/xdm/Xaccess` and uncomment the line that has *#any host can get a login window*
* If you don't want a graphical login on the Raspberry Pi HDMI port sudo Edit `/etc/X11/xdm/Xservers` and comment out the line `:0 local /usr/bin/X :0 vt7 -nolisten tcp`. 
* `sudo systemctl restart xdm.service` to restart the service with the new settings

The edit-xdm-config script (on this github) makes the above modifications automatically.

**Note**: If your system has lightdm installed and you want to switch to xdm (or vice versa), the package installer will ask you which one you want to use. After completing the installation, reboot the system for the change to take effect. To switch between the two display managers once they are both installed, use `dpkg-reconfigure lightdm`.

### VNC Service configuration

* A systemd socket/service conifg file pair is required for each VNC port, and each port will implement a single screen resolution. You can easily create the socket/service pair files with the make-systemd-xvnc script (on this github) or you can create your own (but make-systemd-xvnc is MUCH easier).
        * Use the make-systemd-xvnc script (on this github) to easily generate usable configurations
        * To download the prototype files from the bottom of this document (make-systemd-xvnc replaces this)
            * Bash> cd /etc/systemd/system
            * Bash> sudo wget ht<span>tps://</span>raw.githubusercontent.com/gitbls/RPiVNCHowTo/master/xvnc0.socket
            * Bash> sudo wget ht<span>tps://</span>raw.githubusercontent.com/gitbls/RPiVNCHowTo/master/xvnc0%40.service
    * Edit the VNC configuration files as needed. If you used make-systemd-xvnc you should not need to modify them unless you run into a problem.
        * If you need to edit them, sudo Edit the .service files as appropriate, located in /etc/systemd/system
        * Change the resolution in xvnc<span>0@</span>.service as desired. I like my VNC window to be nearly full screen size on my 1900x1200 monitor, so I use 1880x1100, which is the setting in xvnc<span>0@.</span>service. For my 1900x1080 laptop I use 1880x960, which I've put in the file xvnc<span>1@.</span>service (with a corresponding xvnc1.socket file.
        * The filenames for the .socket and the .service file must match, except for the @ in the .service filename.
        * The @ in the filename is important. When a VNC connection is made, a new service is started with the name similar to xvnc0@n-serveripaddr:port-remoteipaddr:port.service. the @ enables that.
* `sudo systemctl daemon-reload` - Must be done when systemd configuration files are modified. If you use make-systemd-xvnc you are given the option of performing the reload and starting the sockets.
* Start the VNC service connections. For each one you've created (e.g., xvnc0 and xvnc1 in this example), start the socket and enable it across reboots:
    * `sudo systemctl start xvnc0.socket` - Starts the socket in the running system
    * `sudo systemctl enable xvnc0.socket` - Sets the socket to start after the system reboots
* **Test** that you can use a VNC client to connect to your server. 

If additional VNC resolutions are needed, copy the xvnc0* files to, for instance, xvnc1.socket and xvnc<span>1@.</span>service. Then, sudo Edit the .socket file to change the socket number (increase by 1 from the previous), and sudo Edit the .service file to change the resolution. After the files are appropriately updated, issue the above sudo systemctl daemon-reload/start/enable command sequence for the new socket.

make-systemd-xvnc looks for existing socket/service pair files, and if found will ask if you want to replace or create additional pairs. Newly-create pairs are numbered following the last xvnc socket/service pair file.

## How does this work?

Once you start the xvnc0.socket, the systemd process listens on the specified TCP port. When a network connect lands on that port, systemd starts the service described by the .service file with the filename corresponding to the socket name. This is quite simple and elegant, and it *just works*.

In the event that it doesn't, see the next section.

## Problem Solving

If things aren't working correctly, the best approach is to look at the system log (with `sudo journalctl`) and also the status of the services with `sudo systemctl status xvnc*`

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
ExecStart=-/usr/bin/Xtigervnc -noreset -inetd -query 127.0.0.1 -geometry 1880x1100 -pn -once -SecurityTypes None
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
