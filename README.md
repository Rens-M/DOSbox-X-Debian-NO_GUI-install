# DOSbox-X-Debian-install
Installation of DOSBox-X on debian without GUI

## Disclaimer
These instructions are how I managed to get it working the way I wanted. I am an amaateur, I have enough knowledge to get things going by useing google. As such this is not to be taken as the way of doing it. I am sure there are better ways and more efficient ways of doing so but this is how I manged.


## Starting point
The start point is a clean install of Debian 12 (Bookworm), with minimal options, during installation deselect `Debian dektop enviroment` and `GNOME`. Select `SSH server` and `standard system utilities`

##Setup sudo
Sudo is not installed with debian as default when installing in this manner so we have to sort that. 
```
~$ su
/home/user/# apt install sudo
/home/user/# /sbin/adduser <user> sudo
/home/user/# systemctl reboot

```
Install git which is needed later and net-tools to make it easy to find the ip when wanting to so some stuff from a terminal.
```
sudo apt install git net-tools
```

## ASLA audio
```
sudo apt-get install alsa-utils 
```
Initialise sound card:
```
sudo alsactl init 
```
Test sound:
```
aplay /usr/share/sounds/alsa/Noise.wav
```
test sound as root:
```
sudo aplay /usr/share/sounds/alsa/Noise.wav
```
I Set Master volume at 100% and control it later from dosbox, as of writing I have not tinkered too much with other sound settings to see if there is more control available.
```
amixer sset Master 100%
```

## DOSBox-X install
I instaled dosbox-x using the [build](https://github.com/joncampbell123/dosbox-x/blob/master/BUILD.md) instructions for ubuntu as found on the DOSBox-X git page https://github.com/joncampbell123/dosbox-x 
There are two libraries quoted that showed non-available for me (`libavdevice58` & `libavcodec-extra58`) I went ahead without them. and I used the `./build-debug-sdl2`option.
After `sudo make install`was completed I removed the git cloned directory.

## Optional Automount CDROM drive
> I spend the last few days getting this to work, which was a pain.... combining a couple of sollution suggestions I found I made it work. (I had to combine them each on their own did not seem to work for me.)
After install from scratch debian added my cdrom drive to fstab as follows yours may be different.
```
/dev/sr0        /media/cdrom0   udf,iso9660 user,noauto     0       0
```
with this as the starting point:
Make a udev rule: `sudo nano /etc/udev/rules.d/99-local-cdrom.rules`
Add the following (ammend for entries as it is/was pupulated in your fstab)
```
KERNEL=="sr0", SUBSYSTEM=="block", ENV{ID_FS_TYPE}=="?*", ACTION=="change", RUN+="/usr/bin/systemd-mount --no-block --automount=yes --collect $devnode /media/cdrom0"
```
Create a systemd-udevd override: `sudo systemctl edit systemd-udevd`
add in the designated space
```
[Service]
MountFlags=shared
```
then installing [udev-media-automount](https://github.com/Ferk/udev-media-automount) per the instructions.
and commenting out the cdrom line from fstab (per documents of udev-media-automount having it in the fstab with no-auto prevents this from autoloading) following up with 
`sudo systemctl daemon-reload && sudo service systemd-udevd --full-restart`

> now when inserting a disk in the drive it gets automatically mounted for me at /media/cdrom0  and I can use that location for mounting in DOSBox without havaing to drop out of dosbox and manually mount the disc.



## Running DOSBox-X
> From here I could run `dosbox-x`but doing so (for me at least on every try including trying on ubuntu server) gives no accerss to keyboard and mouse, so I have to run `sudo dosbox-x` this works well it just places the config file then in `/root/.config/dosbox-x/dosbox-x-[version].conf unless using the -conf pointer in the command line. 

## Autologin and autosart
### Autologin
For the system to auto login:
```
sudo nano /etc/systemd/logind.conf
```
and add `NAutoVTs=1` at the bottom of the file

then create an override file
```
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d/
```
```
sudo nano /etc/systemd/system/getty@tty1.service.d/override.conf
```
add the config make sure to replaace <user> with your system user
```
[Service]
  ExecStart=
  ExecStart=-/sbin/agetty --noclear --autologin <user> %I 38400 linux

```

### Autostart

> I tried many ways of getting dosbox to autostart I staarted off with running it as a service but found out it echos commands back to the system,  by typing the username [enter] and then password [enter] linux logged in underneath.  To avoid potential issued I could imagining happeneing I gave up on the service and went the following route.



I got rid of the need for password use to run sudo so the system would not ask for it when using it later in a script. I did this by:
```
EDITOR=nano sudo visudo
```
and then ammending the file as follows (replace user with your user on the system) 
```
# User privilege specification
root    ALL=(ALL:ALL) ALL      # (note this line is existing)
user   ALL=(ALL) NOPASSWD:ALL

```
and then removing the user from the sudo group
```
sudo gpasswd -d <user> sudo
```

Create the startup script
```
nano .dosbox_x_start.sh
```
```
#!/bin/bash
sudo dosbox-x -conf /home/<user>/.config/dosbox-x/dosbox-x-[version].conf
```
```
sudo chmod 755 ./.dosbox_x_start.sh
```
> note I user the -conf pointer to make it easier to change the config manually in terminal without having to switch to SU to access the copy that would otherwise be in the /root/.config/dosbox-x location.

Add the startup script to `.profile` I used a funtion that prevents it from running when logging in remotely via ssh to make changes to the system, so at the bottom of the file add:
```
# run dosbox startscript on local login
if [ -z "$SSH_CONNECTION" ]; then
    echo "Starting dosbox-X."
    ~/.dosbox_x_start.sh
fi

```

## Samba
> The whole reason why I went with this DOSBox-x setup rather than using genuine DOS or FreeDos is because I do not know enough to get those shared over the network to loads files to the system easy.  If anyone knows how to make DOS or FreeDos a CIFS sever I'd love to know for my next project.

```
sudo apt install samba
```
```
sudo nano /etc/samba/smb.conf
```
At the bottom of the file add:
> I went with the include file option but you could include the config directly in this file too
```
include = /etc/samba/<filename>.conf
```
```
sudo nano /etc/samba/<filename>.conf
```
add the config, this just points to the normal user folder in which I created folders later mounted in dosbox-x, when running dosbox-x as sudo, any folders you create from within dosbox e.g. `C:\>md games` would be owned by root in the system. To solve access permission issues when accessing the share I forced root user in that. 
```
[dosbox-x]
  comment = Dosbox PC folders for easy upload
  path = /home/<user>/
  read only = no
  browseable = yes
  # Dosbox writes as root to solve access errors we force root
  force user = root
  # Dosbox writes as root to solve access errors we force root 
  force group = root    

```
Set up a Samba password for the user (the user added must be a system user as well), remember to replace <user> with your username.
```
sudo smbpasswd -a <user>
```
```
sudo service smbd restart
```
Now you can access the drive via network share with the credentials you have given samba

## Reboot and enjoy.
Rebooting the system now should autologin and then start DOSBox-X.  Configure DOSBox-X to your liking in normal manner.





