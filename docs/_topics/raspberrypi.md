---
title: Using Raspberry Pi
---

# {{ page.title }}

## Pi Documentation
* <https://www.raspberrypi.org/documentation/>.
* <https://www.raspberrypi.org/documentation/configuration/>.
* <https://pifi.imti.co/>.
* <https://youtu.be/qeHpXVUwI08>
* <https://youtu.be/RlgLIr2gZFg>

## IOT
If we want to make an IOT with Pi, we will need to setup a headless pi first. We will use raspberry pi zero W since it has built-in wireless which
can be used to network for development as well as connecting the device to the internet without additional hardware.  

## Setup for development
We will use a PC to do code editing and run code to test during development. We will setup the wifi to connect the pi to a network that the PC is connected to.

### Setup for headless wifi and USB networking
First burn the minimal boot image to SD card using the PC. After the image is prepared,  
take out the and reinsert the SD card in the PC to make the new filesystem visible. Now go to the root directory
of the SD.

First we will setup the wifi networking. 
Create following two files in the disk image root directory.
1. ``wpa_supplicant.conf``.
2. ``ssh``.

The ``wpa_supplicant.conf`` file should contain the following content,

```auto
country=US
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
update_config=1
network={
	ssid="Wifi-Network-Name"
	psk="Wifi-Network-Password"
	key_mgmt=WPA-PSK
}
```

The ``ssh`` file should be empty. This will enable incoming ssh connections into pi.

These two files will setup the config during boot and then will be deleted after boot, but we will not boot it yet.

Next we will setup USB networking. 
Use a text editor to edit ``config.txt`` in the same directory.
Go to the bottom and add,

```auto
dtoverlay=dwc2
```
Save the ``config.txt``.

Now edit ``cmdline.txt`` file and insert after ``rootwait`` (the last word on the first line) add a space and then ``modules-load=dwc2,g_ether``. Note that this line is a very long line. 

```auto
... rootwait modules-load=dwc2,g_ether ...
```

Save the ``cmdline.txt``

Insert the SD in pi. Now we can use 
USB networking to ssh into pi for development. First make sure that there is
**no power cable is connected to the Pi**. Simply plug in the Pi USB OTG port to a PC or laptop with a cable. PC will recognize the
Pi device power it thru the cable. After the boot completed, you can ssh into ``pi@raspberrypi.local`` using the default password ``raspberry``.

You can also ssh thru wifi. Detach the cable from computer.
Plug in the power cable to the power port 
and turn power on. After the boot completed, 
we can connect to headless pi thru ssh from the computer on the wifi network.

You should change the default password on this first boot to something else. 

### Secure the ssh
Now that you are connected to pi via ssh, 
it is best to setup key-based authentication instead of using password for ssh at this point to make it more secure. 
Key pairs are two cryptographically secure keys and extremely difficult to break. One is private, and one is public. These keys are stored by default in the ``.ssh`` folder in your home directory on the PC. The private key will be called ``id_rsa`` and the associated public key will be called ``id_rsa.pub``. If you don't have those files already, simply use ``ssh-keygen`` command to generate them.

We will need to copy the public key to Raspberry Pi.
Run the following commands on Pi over ssh,

```bash
mkdir -p ~/.ssh
echo >> ~/.ssh/authorized_keys
```

Next, we will put the key in ``~/.ssh/authorized_keys`` using nano,
```bash
nano ~/.ssh/authorized_keys
```

``id_rsa.pub`` is just a text file, open it on your PC and copy the entire
content and paste it in nano at the end of the ``~/.ssh/authorized_keys`` and save.  
Now log in again using ssh using another terminal. If it didn't ask for password then
we have successfully set up the keys. Now we can safely disable password logins, so that all authentication is done by only the key pairs without locking us out.

On pi we will change ``/etc/ssh/sshd_config``,
```bash
sudo nano /etc/ssh/sshd_config
```
There are three lines that need to be changed to ``no``, if they are not set that way already,

```auto
ChallengeResponseAuthentication no
PasswordAuthentication no
UsePAM no
```
Save the file and either restart the ssh system with ``sudo service ssh reload`` or reboot.

### Create a Samba share
We will use code editor on the PC to edit files directly on the pi. We will 
install Samba to do this.
Samba is available in Raspbian’s standard software repositories. We’re going to update our repository index, make sure our operating system is fully updated, and install Samba using apt-get. In ssh terminal and type:

```bash
sudo apt-get update
sudo apt-get upgrade
sudo apt-get install samba samba-common-bin
```
We’re going to create a dedicated shared directory on our Pi’s SD. 
```
sudo mkdir -m 1777 /home/pi/devcode
```
This command sets the sticky bit (1) to help prevent the directory from being accidentally deleted and gives read/write/execute (777) permissions on it.

Edit Samba’s config files to make the file share visible to the Windows PCs on the network.
```
sudo cp /etc/samba/smb.conf /etc/samba/smb.conf.orig
sudo nano /etc/samba/smb.conf
```
In our example, you’ll need to add the following entry:

```
[devcode]
Comment = Pi shared folder
path = /home/pi/devcode
writeable=Yes
create mask=0750
directory mask=0750
valid users = pi 
public=no
```

It is also best to comment out anything that you don't need to use such ``printers`` or ``home`` sections.

Before we start the server, you’ll want to set a Samba password - this is not the same as your standard default password (raspberry), but there’s no harm in reusing this if you want to, as this is a low-security, local network project.

```
sudo smbpasswd -a pi
```
Then set a password as prompted.

Finally, let’s restart Samba:
```
sudo /etc/init.d/smbd restart
```
From now on, Samba will start automatically whenever you power on your Pi. From you windows PC file explorer you can connect to ``\\raspberrypi`` and map ``devcode``
to a drive letter. You can do rest of your development using popular ``vscode`` or any other editor from your PC on the newly created drive.

## Configure IOT setup mechanism by user
If we build IOT device, it needs to be configured. For example the user needs to setup the wifi connection information so that it can be connected to internet.
The question is, how do we set it up with a PC or cell phone and input those
setup?

The basic strategy is to setup up a web page that collects the configuration data. We will need to setup a web server first to produce the interface.

Once done, we can scan wifi networks from pi to get all the available access points. Fot instance we can following shell command to scan and return the result.  

```bash
sudo iw wlan0 scan
```

We can use returned info in the configuration webpage 
for the user to select a
a wifi connection and provide password. 

### Installing the web server

We will be using instructions from [Installing Lighttpd with Python CGI support](https://mike632t.wordpress.com/2013/09/21/installing-lighttpd-with-python-cgi-support/#:~:text=Lighttpd%20is%20a%20lightweight%20web,such%20as%20the%20Raspberry%20Pi.).

Install ``lighthttpd`` web server

```bash
sudo apt-get install lighttpd
```

Create a directory for the content

```bash
mkdir -p /home/pi/devcode/httpd/cgi-bin
cp -Rv /var/www/* /home/pi/devcode/httpd
sudo chown -R www-data /home/pi/devcode/httpd
sudo chgrp -R www-data /home/pi/devcode/httpd
find /home/pi/devcode/httpd -type d -exec sudo chmod g+ws {} \;
sudo adduser pi www-data
```

We will edit ``/etc/lighttpd/lighttpd.conf`` with ``nano`` to update server configuration.
```bash
sudo nano /etc/lighttpd/lighttpd.conf
```

We will change document root in ``/etc/lighttpd/lighttpd.conf``,

```auto
server.document-root        = "/home/pi/devcode/httpd/html"
```

We will Append following to the end of ``/etc/lighttpd/lighttpd.conf`` to enable cgi,

```auto
server.modules += ( "mod_cgi" )
static-file.exclude-extensions += ( ".py", ".py3", ".sh", )
$HTTP["url"] =~ "^/cgi-bin/" {
	alias.url += ( "/cgi-bin/" => "/home/pi/devcode/httpd/cgi-bin/" )
	cgi.assign = (
		".py"  => "/usr/bin/python",
		".py3" => "/usr/bin/python3",
		".pl"  => "/usr/bin/perl",
		".sh"  => "/bin/sh",
	)
}
```

Restart the server
```bash
service lighttpd restart
```

Now we can put static contents in ``httpd/html`` directory and all the handlers
in ``httpd/cgi-bin`` directory. Go ahead, test the server from a web browser with some static content and cgi.

### Using privileged commands in CGI
The web server cgi scripts may need to run commands with root permission. This can be allowed by
updating sudo permission for the server for specific commands. For instance, we can scan the wifi networks using ``/sbin/iw`` command
running with root permission. You can edit the permission by running,

```bash
sudo visudo
```        
This will bring up ``nano`` with the permission file. Add following line at the end of file,

```auto
%www-data  ALL=NOPASSWD: /sbin/iw
```
Now save the file. You can add more than one commands in comma separated paths if needed. Check documentation for ``visudo``.

### Idea 1: Configure via wifi
Set it up initially as a wifi access point on power up. 
Then use it to setup up the configuration.

Perhaps We can run both ap and client at the same time? Or a reset switch to select the mode. Or we can use some other algorithmic way turn on the access point. We can use captive portal to show the user interface.
* <https://www.raspberrypi.org/forums/viewtopic.php?t=211542>.
* <https://serverfault.com/questions/869857/systemd-how-to-selectively-disable-wpa-supplicant-for-a-specific-wlan-interface>.
* <https://pifi.imti.co/>.
* <https://en.wikipedia.org/wiki/Captive_portal>.

Check an [implementation](#idea1-impl). I haven't tested this yet. 

### Idea 2: Configure via bluetooth
make pi a bluetooth device, connect your phone to it with an app the should display user interface and send the info to the device to get it configured.

> is it possible that device will send a html page while the bt connections act as network connection? probably not a whole lot different from idea 1 if we do that.

I haven't tested this idea yet.

### Idea 3: Configure via USB
Connect the device with a usb cable to a computer of phone, again the same concept a user interface shows up to configure.
* [HEADLESS PI ZERO SSH ACCESS OVER USB (WINDOWS)](https://desertbot.io/blog/headless-pi-zero-ssh-access-over-usb-windows).
* [Raspberry pi boot overlays](https://github.com/raspberrypi/firmware/blob/master/boot/overlays/README).
* [Go Go Gadget Pi Zero](https://learn.adafruit.com/turning-your-raspberry-pi-zero-into-a-usb-gadget).
* [RASPBERRY PI ZERO USB/ETHERNET GADGET TUTORIAL](https://www.circuitbasics.com/raspberry-pi-zero-ethernet-gadget/#:~:text=The%20Raspberry%20Pi%20Zero's%20small,is%20shared%20over%20USB%20too.).

Note that, phone has a usb otg connector, and so is pi zero. Both will be in gadget mode. To connect to a phone we will need a special cable which is not desired but possible.

However, let's explore the idea of Pi as a device connected to PC or laptop host.
Pi has usb otg, means that it can be either a host or it can be a device.
We can connect them with a cable and setup Pi as a Ethernet gadget. Then the configuration webpage will be visible from PC browser. This seems to
be most straight forward way since our Pi is already setup for
USB networking.

Make sure that the **power cable is removed from the Pi**. Simply plug in the Pi USB OTG port to a PC or laptop. PC will power and recognize the
Pi device. At this point you can open a browser and browse to 
``http://raspberrypi.local`` and the web page will be displayed.

## <a name="idea1-impl">Raspberry pi as Access Point and Wifi client

This is an example of how the *idea 1* can be implemented. This was collected from the tutorials found on internet. 

> It is based on IOT wifi's [solution](https://pifi.imti.co/), but I wanted to use a language other than Go to manage my wifi connections, so all changes are within the standard Raspbian Stretch OS.

These steps are (as best as I can remember) in the order that I did them in:

### 1. Update system
Run apt-get update and upgrade to make sure you have the latest and greatest.
```
sudo apt-get update
sudo apt-get upgrade
```

This may take a while depending on connection speed.

### 2. Install hostapd and dnsmasq
Install the hostapd access point daemon and the dnsmasq dhcp service.
```bash
sudo apt-get install hostapd dnsmasq
```

### 3. Edit configuration files
Here we need to edit the config files for dhcpcd, hostapd, and dnsmasq so that they all play nice together. We **do NOT**, as in past implementations, make any edits to the ``/etc/network/interfaces`` file. If you do it can cause problems, check tutorial notes [here](https://raspberrypi.stackexchange.com/questions/37920/how-do-i-set-up-networking-wifi-static-ip-address/37921#37921).

Edit ``/etc/dhcpcd.conf``
```
interface uap0
	static ip_address=192.168.50.1/24
    nohook wpa_supplicant
```
This sets up a static IP address on the uap0 interface that we will set up in the startup script. The nohook line prevents the 10-wpa-supplicant hook from running wpa-supplicant on this interface.

Replace ``/etc/dnsmasq.conf``
Move the dnsmasq original file to save a copy of the quite useful example, you may even want to use some of the RPi-specific lines at the end. I did not test my solution with those.
```
sudo mv /etc/dnsmasq.conf /etc/dnsmasq.conf.orig
```
Create a new ``/etc/dnsmasq.conf`` and add the following to it:
```
interface=lo,uap0    # Use interfaces lo and uap0
bind-interfaces      # Bind to the interfaces
server=8.8.8.8       # Forward DNS requests to Google DNS
domain-needed        # Don't forward short names
bogus-priv           # Never forward addresses in the non-routed address spaces

# Assign IP addresses between 192.168.70.50 and 192.168.70.150
# with a 12-hour lease time
dhcp-range=192.168.70.50,192.168.70.150,12h

# The above address range is totally arbitrary; use your own.
```

Create file ``/etc/hostapd/hostapd.conf`` and add the following:
(Feel free to delete the commented out lines)

```
# Set the channel (frequency) of the host access point
channel=1

# Set the SSID broadcast by your access point (replace with your own, of course)
ssid=IOT-Config

# This sets the passphrase for your access point (again, use your own)
wpa_passphrase=passwordBetween8and64charactersLong

# This is the name of the WiFi interface we configured above
interface=uap0

# Use the 2.4GHz band
# (untested: ag mode to get 5GHz band)
hw_mode=g

# Accept all MAC addresses
macaddr_acl=0

# Use WPA authentication
auth_algs=1

# Require clients to know the network name
ignore_broadcast_ssid=0

# Use WPA2
wpa=2

# Use a pre-shared key
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
driver=nl80211

# I commented out the lines below in my implementation, but I kept them here for reference.

# Enable WMM
#wmm_enabled=1

# Enable 40MHz channels with 20ns guard interval
#ht_capab=[HT40][SHORT-GI-20][DSSS_CCK-40]
```

> Note: The channel written here MUST match the channel of the wifi that you connect to in client mode (via wpa-supplicant). If the channels for your AP and STA mode services do not match, then one or both of them will not run. This is because there is only one physical antenna. It cannot cover two channels at once.

Edit file ``/etc/default/hostapd`` and uncomment ``DAEMON_CONF`` line. Add the following,

```
DAEMON_CONF="/etc/hostapd/hostapd.conf"
```

### 4. Create startup script
Add a new file ``/usr/local/bin/wifistart`` (or whatever name you like best), and add the following to it:
```
#!/bin/bash

# Redundant stops to make sure services are not running
echo "Stopping network services (if running)..."
systemctl stop hostapd.service
systemctl stop dnsmasq.service
systemctl stop dhcpcd.service

# Make sure no uap0 interface exists (this generates an error; we could probably use an if statement to check if it exists first)
echo "Removing uap0 interface..."
iw dev uap0 del

# Add uap0 interface (this is dependent on the wireless interface being called wlan0, which it may not be in Stretch)
echo "Adding uap0 interface..."
iw dev wlan0 interface add uap0 type __ap

# Modify iptables (these can probably be saved using iptables-persistent if desired)
echo "IPV4 forwarding: setting..."
sysctl net.ipv4.ip_forward=1
echo "Editing IP tables..."
iptables -t nat -A POSTROUTING -s 192.168.70.0/24 ! -d 192.168.70.0/24 -j MASQUERADE

# Bring up uap0 interface. Commented out line may be a possible alternative to using dhcpcd.conf to set up the IP address.
#ifconfig uap0 192.168.70.1 netmask 255.255.255.0 broadcast 192.168.70.255
ifconfig uap0 up

# Start hostapd. 10-second sleep avoids some race condition, apparently. It may not need to be that long. (?) 
echo "Starting hostapd service..."
systemctl start hostapd.service
sleep 10

# Start dhcpcd. Again, a 5-second sleep
echo "Starting dhcpcd service..."
systemctl start dhcpcd.service
sleep 5

echo "Starting dnsmasq service..."
systemctl start dnsmasq.service
echo "wifistart DONE"
```

There are other and better ways of automating this startup process, which I adapted from IOT wifi's code [here](https://github.com/cjimti/iotwifi). This demonstrates the basic functionality in a simple script.

### 5. Edit rc.local system script
There are other ways of doing this, including creating a daemon that can be used by systemctl, which I would recommend doing if you want something that will restart if it fails. Adafruit has a simple write-up on that [here](https://learn.adafruit.com/running-programs-automatically-on-your-tiny-computer/systemd-writing-and-enabling-a-service). I used ``rc.local`` for simplicity here.

Add the following to your ``/etc/rc.local`` script above the exit 0 line. Note the spacing between ``/bin/bash`` and ``/usr/local/bin/wifistart``.
```
/bin/bash /usr/local/bin/wifistart
```
### 6. Disable regular network services
The wifistart script handles starting up network services in a certain order and time frame. Disabling them here makes sure things are not run at system startup.
```
sudo systemctl stop hostapd
sudo systemctl stop dnsmasq
sudo systemctl stop dhcpcd
sudo systemctl disable hostapd
sudo systemctl disable dnsmasq
sudo systemctl disable dhcpcd
```
### 7. Reboot
```
sudo reboot
```

If you want to test the code directly and view the output, just run
```
sudo /usr/local/bin/wifistart
```
from the terminal after commenting out the ``wifistart`` script line in ``rc.local``.

## Preparing for distribution.

Back up the sdio image from dev SD card first.

Now we have to make sure the image has disabled ssh, and samba and any other services not needed on the deployed device by running some kind of shell script.
now the has become the production SD and ready for imaging.

Save the sdio image from dev SD card first. This will be the boot image to be downloaded.

We will put minimal code on the SD, something similar to ``noobs`` (New Out of Box Software), the boot image should be downloaded and prepared after initial boot by the user during configuration.
> Check noobs source code.

For application where production SD image is small, there will be no benefit using **NOOBS** strategy. In those case, fully loaded SD should be fine. 

* Imaging sdio source code <https://github.com/raspberrypi/rpi-imager>
* <https://github.com/raspberrypi/noobs>

> todo: check how to use docker container in pi

## Wifi related links
* <https://www.raspberrypi.org/forums/viewtopic.php?t=179387>
* <https://www.raspberrypi.org/documentation/configuration/wireless/wireless-cli.md>
* <https://www.tecmint.com/set-system-locales-in-linux/>
* <https://www.debian.org/doc/manuals/debian-reference/ch08.en.html>


## Some random setup stuff
* ``timedatectl list-timezones`` provides list of all timezones
* <https://github.com/eggert/tz> all timezones
* raspi-config <https://github.com/RPi-Distro/raspi-config>
* list of 2 letter country codes <https://www.iso.org/obp/ui/#search>
* list of flags <https://github.com/lipis/flag-icon-css>
* language codes <https://www.loc.gov/standards/iso639-2/php/code_list.php>
* select from a list <https://svelte.dev/tutorial/select-bindings>

## Samba WINS doesnt make is discoverable in windows 10
WSD is missing from samba. samba only supports netbios. This WSD server written in python will, make the device discoverable.  
* <https://github.com/christgau/wsdd>