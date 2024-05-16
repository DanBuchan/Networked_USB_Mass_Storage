# Networked_USB_Mass_Storage

This is a little project to turn a raspberry pi in to a USB mass storage device which stores it's data on the network via SAMBA. For now, the following is just a HOW TO achieve this. Soon I will make this in to a shell script you can run on your pi to "auto" configure everything.

I did this mostly to help retro computer folks who are using things like gotek floppy emulators so they can have a means of keeping their floppy images on some centrally adminsitered place on the network. So it kind of gives you a way to give a way to pseudo-network access any retro computer that will take a gotek. But really this is a generic solution for instance, if you have a TV that will play MP4s from USB stick, then you could use this to store your films on your NAS. 

## Preliminary set up

You have to have a raspberry pi that can be configured in to USB gadget mode. Not all can but all of the boards that have only a single USB port can be. Double check the docs for your pi. I was using a Pi 3A+

### RPi OS install

Get the latest Raspberry [PI Imager](https://www.raspberrypi.com/software/). Pick your OS but importantly set you wifi SSID and password and enable SSH in the pre-install dialogue

### Ensure you can SSH to the machine

As we'll be setting up the RPi in USB Gadget mode as a USB Mass Storage device the Pi will not longer be able to act as a USB host, this means once complete the USB port will no longer be able to read a keyboard or mouse, and in turn in the future the only way in to configure and use the RPi will be via SSH over WiFI. You should boot the pi with the OS and ensure you can connect to the pi over SSH. On windows you'll need a terminal client like [PuTTY](https://www.putty.org/). In linux and mac you can just use ssh via a terminal.

## Configure some SAMBA network storage

1. Create some folder on your windows PC or SAMBA host (like a home NAS). I called mine `/smb_file_share/`
2. Set the folder to allow sharing on the network. In a windows PC open the folder properties and under sharing allow access to everyone with read/write permission.
3. Write down the SAMBA location of the drive, mine was something like `smb://DESKTOP-TLSJI9N/smb_file_share`

## Configure the pi

There are basically three things we need to do, some preliminary OS changes, set the pi to automount the shared SAMBA dir when it boots, and set the pi to boot configured as a USB Mass Storage device

### Preliminary config

#### Update SWAP size

For some reason the RPi OS has a zero sized SWAP partition which makes it a ballache to do some things with them. Assuming your Pi's SDCard is a sensible size (more than a couple of GBs) then 1GB of swap is probably sensible. Follow this handy guide: [Change Swap size](https://ppfeufer.de/raspberry-pi-change-swap-size/)

#### Set to boot to non-graphical mode

You'll not need to connect the pi to a display and the pi will boot WAY FASTE if you turn off the GUI/Desktop. You can set it to boot to non-graphical. Boot the pi and in a terminal (maybe even via ssh as above) type:

``` bash
sudo raspi-config
```

Then find the setting to change the boot mode and set it to `B2 console autologin`

### Configuring the device

#### Set up SAMBA mount point

First install some things so SAMBA access will work. Then make a directory to be the mount point for the SAMBA share. Then create a SAMBA credentials file

``` bash
sudo apt-get install samba-common samba-common-bin smbclient cifs-utils
sudo mkdir /mnt/file_share/
sudo vi ~/.smbcreds
sudo chmod 600 ~/.smbcreds
```
now open the file .smbcreds and add your SAMBA username and password in the following format
 
```
username=[USERNAME_GOES_HERE]
password=[PASSWORD_GOES_HERE]
```

Next we must update the filesystem table, use the command:

```
sudo vi /etc/fstab
```
To the bottom of the table add the following line, taking care to add the correct SAMBA share address and ensure the correct path to the .smbcreds file

```
//DESKTOP-TLSJI9N/smb_file_share /mnt/file_share cifs credentials=/home/[RASPBERRYPI_USER_ID]/.smbcreds 0 0
```

Then reload all system deamons with the command

``` bash
sudo systemctl daemon-reload
```

Now you can reboot the pi and it should automatically mount the SAMBA share directory to `/mnt/file_share/` on the pi. You can test this out by making a file on the SAMBA share and you should see it appear in `/mnt/file_share/` on the pi. And conversely you can make a file in `/mnt/file_share/` on the pi and it'll show up in the SAMBA host. Assuming all that works. HURRAH!

#### Set up Pi as USB mass storage

Most of the following info was gleaned from [https://github.com/thagrol/Guides](https://github.com/thagrol/Guides), in the Mass Storage Gadget pdf. You can find more info there but basically we're going to use the pi to create a new block store file (using `dd`), this will act as the file system for the USB Mass storage device and we're going to keep that block store in the SAMBA dir. Once that exists we can configure the PI as a USB mass storage gadget which uses that block store to store all it's files. 

So... on the pi in a terminal; First make the block store in the SAMBA dir. I'm just going to make a 1Gb block store/file system but you can adjust as you see fit. This makes a 1GB file in the samba share called `files.img`

``` bash
cd /mnt/file_share
sudo dd if=/dev/zero of=files.img bs=1M count=2K
```

Next, add this to be mounted whenever the pi boots

``` bash
sudo crontab -e
```
To the bottom of the crontab table add the following line

```
@reboot /sbin/modprobe g_mass_storage file=/mnt/file_share/files.img
```
Now edit the Pi firmware config so it boots in USB gadget mode

``` bash
sudo vi /boot/firmware/config.txt
```

Scroll to the bottom of this file and add the following to the [all] section

```
[all]
dtoverlay=dwc2,dr_mode=peripheral
```

Now you can reboot. The pi should boot, mount the samba share and over the USB interface it should pretend to be a USB Mass storage device which is storing all it's file in the `files.img` file that is hosted on the network

### Format networked USB device

You now have a networked USB stick but the file store needs a file system, so you need to format it. There are many ways to do this. The easiest by far is to plug the pi in to a windows PC and format the "disk" (strictly speaking you're formatting the `files.img` block file). In windows attach the pi, go to the disk manager and assign the device a drive letter, then format as FAT32.

You can achieve the equivalent pretty easily in Mac and Linux too.

## Now use your Networked USB stick

Ok, you need to think about boot order when using this. Firstly the device hosting you SAMBA shared directory on and the directory available BEFORE you boot the RPi. Once the Pi is booted and has mounted the SAMBA share, only then should you plug the Pi in to the USB of a new USB host. 

1. Boot and/or ensure you SAMBA host is running and accessible
2. Boot your RPi
3. Boot the USB host
4. Now plug the PPi in to the USB Host

It is critical that 1 happens before 2, the others ought to be fine to happen in any order.

Assuming all has gone well you can now use your network attached USB Mass Storage Device but there are two ways to get files "on" to the device

### Like a regular USB stick

Just plug the stick in to any machine/hosts and copy files over as you would any USB stick. Hurrah

### Mount `files.img` directly.

Any computer that can open the shared SAMBA directroy can mount the `files.img` file. In windows you'll need someting like [Imdisk](https://sourceforge.net/projects/imdisk-toolkit/) and then use mount img tool it provides. It'll add a drive that represents the contents  of `files.img` file and you can copy files to that drive as you would any drive. And nicely they'll show up on whatever computer the RPi is connected to. This is kind of the whole point of this, that you can centrally administer the contents of `files.img` and any computer the Pi is attached to can then see the changes.

One thing to consider here is just to have `files.img` always mounted by the machine(s) you use to centrally administer the USB file store.

## Gotek Specific Notes

### Flashfloppy Firmware

As this is just a USB mass storage device don't forget to put the (flashfloppy firmware)[https://github.com/keirf/flashfloppy] files in the root of the USB/`files.img` (e.g. AUTOBOOT.HFE, HSXSDFE.CFG, etc).  

### Data Only USB Cable

So I found that my gotek was not fully electrically isolated. Clearly there are some diodes that should be in place that are not. With my RPi powered and connected to the gotek it was able to turn on the LED circuit in my Amiga even when the Amiga were not powered up. Concerning to say the least. I guess everyone who designed the gotek assumed you'd only attach a passive USB stick, and not some powered device like a RPi. 

Personally I don't want power flowing in to my precious Amiga from multiple sources.

To remedy this you should buy or make a USB data-only cable. I could not find any USB A data-only cables to buy, so I made one by buying a 15cm USB cable and disconnecting the power pins. To do this, carefully slice the cable open, either cut or push aside the copper ground/draining threads, then very carefully unwrap any protective wrapping. You should find 4 tiny wires; 2 for power (black and red) and 2 for data (white and green). Cut either one of the power wires.

# TODOS

1. Write shell script to just do all this in one go
2. Swtich to libcomposite as g_mod_probe is deprecated now
