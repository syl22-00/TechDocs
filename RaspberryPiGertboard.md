Raspberry Pi and Gertboard
==========================

Here are notes I am taking while experimenting with the Raspberry Pi device and the Gertboard.

# 1. Purchase

* Raspberry Pi model B, [275 RMB from Taobao](http://item.taobao.com/item.htm?spm=2013.1.0.0.r2GPR7&scm=1007.77.0.0&id=16188941402&pvid=ee918e46-a256-4b00-8328-58d5e2360387&ad_id=&am_id=&cm_id=&pm_id=).
* Gertboard fully assembled, [448 RMB from Taobao](http://item.taobao.com/item.htm?spm=a230r.1.14.3.XRNxHx&id=16522262978).
* SD card, 8 GB, class 10, 75 RMB from a shop.

# 2. Preparing the OS

* Download the [latest Raspbian image](http://www.raspberrypi.org/downloads).
* Write it on the SD card:

    $ dd bs=4M if=2013-05-25-wheezy-raspbian.img of=/dev/sdb # If the SD card is on /dev/sdb

# 3. Connection

To set up the Raspberry Pi:

* HDMI output to TV,
* SD card in slot,
* Ethernet cable to Ethernet hub (with DHCP),
* USB Keyboard,
* An at last, the micro USB power supply.

At first boot, it launches raspi-config, a small utility to set things up.

* In advanced settings, start SSH,
* Exit raspi-conf,
* Find out the IP address with `ifconfig`.

# 4. Initial set-up

You can now connect from a real computer through SSH (pi/raspberry), and you'd likely want to change the following things:

Using raspi-config:

* Expand the partitions to take the whole space,
* Change password.

Then change IP address from DHCP to fixed. The DHCP server was set up to start from `192.168.0.99`, so we assign from `192.168.0.98` and downwards.

I have an old Ralink Wifi USB device, and in order to be able to use either the Ethernet connection or the Wifi, whathever is connected, I had to do the followinf, thanks to [this blog post](http://www.janwagemakers.be/wordpress/?p=276). Here is the content of my `/etc/network/interfaces`:

    auto lo
    iface lo inet loopback
    
    allow-hotplug wlan0
    iface wlan0 inet static
    	pre-up	   ifconfig eth0 down
    	address 192.168.0.97
    	netmask 255.255.255.0
    	gateway 192.168.0.1
    	wpa-driver wext
    	wpa-ssid my_ssid
    	wpa-psk "my key"
    	post-down  ifconfig eth0 up
    
    dns-nameservers 192.168.0.1 8.8.8.8 4.2.2.1
    
    iface eth0 inet static
    	pre-up	ifdown wlan0
    	address 192.168.0.98
    	netmask 255.255.255.0
    	gateway 192.168.0.1
    	post-down ifup wlan0
    
    #dns-domain yourdomain.com
    dns-nameservers 192.168.0.1 8.8.8.8 4.2.2.1


