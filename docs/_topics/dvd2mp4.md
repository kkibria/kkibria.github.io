---
title: Converting old DVDs into mp4
---

# {{ page.title }}

Old DVDs take space and bulky. I have tons of those that I accumulated over the years and running out of space in my cabinet. This will be an ongoing project for me to turn them into mp4 so that I can store them in a hard drive and view them from my TV.

You will need first install the following software before ripping a DVD:

* [Handbrake](https://handbrake.fr/)
* [Ubuntu Restricted Extras](https://www.lifewire.com/manage-fonts-in-linux-4176886)
* Libdvd-pkg

You can find Handbrake in the default Ubuntu repositories, but there's a decent chance that the package will be fairly outdated. Thankfully, the Handbrake developers maintain an [official Ubuntu PPA](https://www.lifewire.com/updated-software-for-ubuntu-with-ppas-2202103).

Begin by opening a terminal window and typing the following command to add the PPA to your system.
```
sudo add-apt-repository ppa:stebbins/handbrake-releases
```
Now, update your package database, and install Handbrake.

```
sudo apt update
sudo apt install handbrake-gtk
```
This will install the video decoding software for converting DVDs to MP4.

Next, type the following command at the terminal prompt to install the restricted extras package. This will install a collection of codecs:
```
sudo apt install ubuntu-restricted-extras
```
During the installation, a blue screen will appear with a license agreement. Press Tab to highlight the option to accept the agreement, and press Enter.

Finally, install the libdvd-pkg to install a library that will let you play DVDs within Ubuntu by entering the following command:

```
sudo apt install libdvd-pkg
```

At the end of the process, you may get a message saying you need to run another apt-get command to continue installing the package. If you get this message, type the following command:

```
sudo dpkg-reconfigure libdvd-pkg
```

## converting to ISO first and then use handbrake

The old linux machine which has a DVD drive is slow. My other machine is fast but there is no DVD drive. So I decided to rip the DVDs to ISO image first and then use the faster machine to turn into mp4. 

Create an ISO disk image from a CD-ROM, DVD or Blu-ray disk.
First get block count. I am using ``/dev/dvd``. In your machine it could be ``/dev/sr0``. Make sure you are using the right device name for your machine.
```
isosize -d 2048 /dev/dvd
```

Now run dd command and display progress bar while using dd command:
```
$ sudo dd if=/dev/dvd of=output.iso bs=2048 count=<blocks> status=progress
```

Combining both in the same script,
```
blocks=$(isosize -d 2048 /dev/dvd)
dd if=/dev/sr0 of=isoimage.iso bs=2048 count=$blocks status=progress
```

Now you can use output.iso for hard disk installation or as a backup copy of CD/DVD media. Please note that dd command is standard UNIX command and you should able to create backup/iso image under any UNIX like operating system.

FYI, you can restore hard disk drive from a previously generated ISO image using the dd command itself using,
```
$ sudo dd if=output.iso of=/dev/dvd bs=4096 conv=noerror
```

## Windows platform DVD decoding in Handbrake
Widonws distribution lacks video decoder for some DVDs. You will see choppy output in sudch case. Download [libdvdcss-2.dll file from VLC](http://download.videolan.org/pub/libdvdcss/1.2.12/) and copy into HandBrake directory. It should resolve the decoder issue.

> I got these instructions from following sources. Read for more details. 
> * [how-to-use-ubuntu-to-convert-dvds-to-mp4](https://www.lifewire.com/how-to-use-ubuntu-to-convert-dvds-to-mp4-4111375).
> * [linux-creating-cd-rom-iso-image](https://www.cyberciti.biz/tips/linux-creating-cd-rom-iso-image.html).
> * [wikipedia dd](https://en.wikipedia.org/wiki/Dd_(Unix)).
> * {{ "https://en.wikipedia.org/wiki/Dd_(Unix)" | split("-,_") | join " "}} 
