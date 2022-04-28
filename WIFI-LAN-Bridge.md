# Wifi to Ethernet bridge (on a RasPi)
*actually, it's not technically a bridge, but it works maybe like you would expect a bridge to work*

download the raspi "lite" image using the raspi disk imager tool and put it on the SD card (this is gonna erase your SD card, i assume that's okay)

***ACTUALLY! you don't need to do some of the following, you can enter it all using the raspi imager tool. fill in username, password, wifi ssid and password, country and locale, click the enable ssh checkbox and finally click the "skip first setup wizard". then you can continue with the guide after the horizontal line (logging in with `ssh`). or you could do it the oldfashioned manual way described below and feel like a hacker :P***

------------------
then you should be able to access a `boot` partition on the SD card from your normal computer.
in the boot partition you'll need to write some text files. 

first: 

a file called userconf.txt containg the following

```
username:$6$K4kkXNMZOgAg0cok$kcxCISzwhTjm37T3YhgMD.rr1axQE6Z9fzq2Jz3AFQOKPTlcuZ6kRueia1u.DRUe26MSSznKqWnbYVgEr0.zG1
```

this creates a user (username) with the password "default password"

next you'll want to save a text file with your wifi credentials, the file name should be `wpa_supplicant.conf` (you might need to change this in explorer with "show file extensions" enabled, because notepad usually gives everything a .txt extension. same applies to ssh below).


```
ctrl_interface=DIR=/var/run/wpa_supplicant GROUP=netdev
country=IE
update_config=1

network={
 ssid="<Name of your wireless LAN>"
 psk="<Password for your wireless LAN>"
}
```

and finally you need to make an empty file named `ssh` in the same place. this enables remote access.

*******************

now you can insert the sd card into the raspi and it should connect automatically to your wifi.

you can connect with ssh, so open up command prompt on your laptop and try:

`ssh <yourusername>@raspberrypi.local` and log in with `default password` or whatever you chose. it might take a few minutes to fully start up first. you could check if it's connected by looking at the list of connected devices in your router web interface. you can also use this to find the IP address, and do `ssh <username>@<IP-address>` instead of the hostname (if the first ssh command didn't work).

okay! you're in. using bash, and some tools we can edit text files:


```bash
sudo nano /etc/dhcpcd.conf
```

add at the bottom of the file: 

```
denyinterfaces eth0
```
to exit nano press ctrl+x and say yes to save modified buffer.

next we need to enable packet forwarding:

```
# open this network config file
sudo nanoenx00e04c534458 /etc/sysctl.conf

# then find this line:
#net.ipv4.ip_forward=1

# and delete the "#" at the beginning of the line and save again with ctrl+x.
```

----------

okay, actually, it gets kinda tricky now...

install git and some other compiler tools:

```
sudo apt install git cmake liblua5.1-0-dev
```

then we need this weird embedded library for routers, download, compile and install it with the following:
```bash
cd
git clone https://git.openwrt.org/project/libubox.git
cd libublox
mkdir build
cd build
cmake .. -DBUILD_EXAMPLES=OFF -DBUILD_LUA=OFF
make 
sudo make install
```

then we can build the tool that uses the aformentioed library:

```bash
cd
git clone https://git.openwrt.org/project/relayd.git
cd relayd
mkdir build
cd build
cmake ..
make
sudo make install
```

next we need to add the library to the compiler search paths:

```
sudo nano /etc/ld.so.conf

# add this line to this file:

include /usr/local/lib
# save again with ctrl+x

# then reload
sudo ldconfig
```

now the command we need should work, try it!

```bash
relayd
```

(it shoud just print out its options)


finally, now create a systemd service, something that will start the wifi/ethernet linking program at boot, and restart it if it crashes...

`sudo nano /etc/systemd/system/reldayd.service`

and paste in the following:

```
[Unit]
Description=Help forwarding from wifi to ethernet
After=network-online.target
Wants=network-online.target

[Service]
User=root
ExecStart=/etc/sbin/relayd -I eth0 -i wlan0 -D -B
Restart=always

[Install]
WantedBy=multi-user.target
```

note: this assumes your interface names are eth0 and wlan0, you can check with the `ip a` command. you should see at least three interfaces that are numbered:
1: lo
2: eth0
3: wlan0

if not, eth0 and wlan0 in the above file will need to be updated.

and enable the service, for starting automatically at each boot:

```
sudo systemctl enable --now relayd
```

aaaand you should be done :D

check by connecting some ethernet device, disable any other network interfaces it might have (like wifi) and make sure you've got an internet connection.

good job!

