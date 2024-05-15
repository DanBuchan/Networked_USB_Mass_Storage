# Networked_USB_Mass_Storage

Little project to turn a raspberry pi USB mass storage device which stores it's data on the network via SAMBA. The following is so far a HOW TO achieve this. Soon I will make this in to a shell script you can run on your pi.

I did this mostly to help retro computer folks using things like gotek floppy emulators to have a means of keeping their floppy images on the network and to give a way to pseudo-network any retro computer that will take a gotek. 

## Preliminary set up

You have to have a raspberry pi that

### RPi OS install

Get the latest Raspberry PI Imager, https://www.raspberrypi.com/software/. Pick your OS but importantly set you wifi SSID and password and enable SSH in the pre-install dialogue

### Ensure you can SSH to the machine

As we'll be setting up the RPi in USB Gadget mode as a USB Mass Storage device the Pi will not longer be able to act as a USB host, soi the USB port will no longer be able to read a keyboard or mouse, so in the future the only way in to configure and use the RPi will be via SSH over WiFI. You should boot the pi with the OS and ensure you can connect to the pi over SSH. On windows you'll need a terminal client like PuTTY, https://www.putty.org/. In linux and mac you can just use ssh via a terminal.

## Configure some SAMBA network storage

1. Create some folder on your windows PC or SAMBA host (like a home NAS). I called mine /smb_adf_share/
2. Set the folder to allow sharing on the network. In a windows PC open the folder properties and under sharing allow access to everyone with read/write permission.
3. Write down the SAMBA location of the drive, mine was something like smb://DESKTOP-TLSJI9N/smb_adf_share

## Configure the pi

### Preliminary config

#### Update SWAP size

For some reason the RPi OS has a zero sized SWAP partition which makes it a ballache to do somethings with. 1GB of swap is probably sensible. Follow this handy guide, https://ppfeufer.de/raspberry-pi-change-swap-size/

#### Set to boot to non-graphical mode

You'll not need to connect the pi to a display so, you can set it to boot to non-graphical. This will make it boot much faster. In a terminal on the pi type

``` bash
sudo raspi-config
```

Then find the setting to change the boot mode and set it to `B2 consol autologin`

### Configuring the device

#### Set up SAMBA mount point

First install some things. Make a directroy to be the mount point for the SAMBA share. Then create a SAMBA credentials file

``` bash
sudo apt-get install samba-common samba-common-bin smbclient cifs-utils
sudo mkdir /mnt/adf_share/
sudo vi ~/.smbcreds
sudo chmod 600 ~/.smbcreds
```
now open the file .smbcreds and add you SAMBA username and password in the following format
 
```
username=[USERNAME_GOES_HERE]
password=[PASSWORD_GOES_HERE]
```

Next we must update the filesystem table, use the command

```
sudo fstab
```
To the bottom of the table add the following line, taking care to add the correct SAMBA share address and ensure the correct path to the .smbcreds file

```
//DESKTOP-TLSJI9N/smb_adf_share /mnt/adf_share cifs credentials=/home/[RASPBERRYPI_USER_ID]/.smbcreds 0 0
```

Then reload all system deamons with the command

``` bash
sudo systemctl daemon-reload
```

Now you can reboot the pi and it should automatically mount the SAMBA share directory to `/mnt/adf_share/` on the pi. You can test this out by making a file on the SAMBA share and you should see it appear in `/mnt/adf_share/` on the pi. And conversely you can make a file in `/mnt/adf_share/` on the pi and it'll show up in the SAMBA host. Assuming all that works. HURRAH!

#### Set up Pi as USB mass storage

Most of the following info was gleaned from https://github.com/thagrol/Guides, in the Mass Storage Gadget pdf. You can find more info there but basically we're going to use the pi to create a new block store (using `dd`), this will act as the file system for the USB Mass storage device and we're going to keep that block store in the SAMBA dir. Once that exists we can configure the PI as a USB mass storage gadget which uses that block store to store all it's files. So on the pi.

First make the block store in the SAMBA dir. I'm just going to make a 1Gb block store/file system but you can adjust as you see fit. This makes a 1GB file in the samba share called `adfs.img`

``` bash
cd /mnt/adf_share
sudo dd if=/dev/zero of=adfs.img bs=1M count=2K
```

Next, add this to be mounted whenever the pi boots

``` bash
sudo crontab -e
```
To the bottom of the crontab table add the following line

```
@reboot /sbin/modprobe g_mass_storage file=/mnt/adf_share/adfs.img
```
Now edit the Pi firmware config so it boots in USB gadget more
``` bash
sudo vi /boot/firmware/config.txt
```
Scroll to the bottom of this file and add the following to the [ALL] section
```
[all]
dtoverlay=dwc2,dr_mode=peripheral
```
Now you can reboot. The pi should boot, mount the samba share and over the USB interface it should pretend to be a USB Mass storage device which is storing all it's file in the `adfs.img` file that is hosted on the network

### Format networked USB device

You now have a networked USB stick but the file store needs a file system, so you need to format it. There are many ways to do this. The easiest by far is to plug the pi in to a windows PC and format the "disk" (strictly speaking you're formatting the `adfs.img` block file). In windows attach the pi, go to the disk manager and assign the device a drive letter, then format as FAT32.

You can achieve the equivalent pretty easily in Mac and Linux too.

## Now use your Networked USB stick

There are two sensible ways to use this

### Just like a regular USB

Just plug the stick in to any machine and copy files over as you would any USB stick. Hurrah

### Mount `adfs.img` directly.

Any computer that can open the shared SAMBA directroy can mount the `adfs.img` file. In windows you'll need someting like https://sourceforge.net/projects/imdisk-toolkit/ and use the mount img tool. Now you'll have a drive that represents the `adfs.img` file and you can copy files to the img across the network and they'll show up on whatever computer the pi is connected to.

## Gotek Specific Notes

### Flashfloppy Firmware

As this is just a USB mass storage device don't forget to put the flashfloppy firmware files in the root of the USB (e.g. AUTOBOOT.HFE, HSXSDFE.CFG, etc), https://github.com/keirf/flashfloppy

### Data Only USB Cable

So I found that my gotek was not fully electrically isolated. Clearly there are some diodes that should be in place that are not. With my RPi powered and connected to the gotek it was able to turn on the LED circuit in my amigas even when the Amigas were not powered up. Concerning to say the least. I assumed that everyone who has designed a gotek assumed you'd only attach a passive USB stick. 

So you should buy or make a USB data-only cable. I could not find any USB A data-only cables to buy so I made one by buying a 15cm USB cable and disconnected the power pins. To do this, carefully slice the cable open, cut any protective copper shielding threads, then very carefully unwrap any protective wrapping. You should find 4 tiny wires; 2 for power and 2 for data. Cut one of the power wires, they should be a pair where one is black and one red, you can cut either.
