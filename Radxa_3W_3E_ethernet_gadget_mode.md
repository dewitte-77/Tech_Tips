# configure Radxa Zero 3W/3E usb network gadget

tested with 2 images:
https://github.com/radxa-build/radxa-zero3/releases/download/rsdk-b1/radxa-zero3_bookworm_kde_b1.output_512.img.xz

https://github.com/radxa-build/radxa-zero3/releases/download/b6/radxa-zero3_debian_bullseye_xfce_b6.img.xz (make sure you update after installation to get the latest version of the radxa-otgutils)

## configuration on the radxa "host"
### activate usb0 device
First add the "peripheral overlay" and activate "Radxa OTG ECM services"

```
sudo rsetup
Overlays -> Yes -> Manage Overlays

Activate "Set OTG port to Peripheral mode" by navigating with the arrow keys and pressing the SPACE bar to select it.
Press TAB key to go to OK and press ENTER

You will get the message
"Selected overlays will be enabled at next boot"
Press ENTER

Go back to main menu and choose

Hardware -> USB OTG Services 
Again navigate with the arrow keys and select "radxa-ecm@fcc00000.usb" by pressing the SPACE bar.
Press TAB, OK

Exit the rsetup menu by choosing CANCEL until you go back to the shell
```

reboot

you should now see a usb0 interface when typing
` ip a

The usb0 device will have a random generated mac address. 

### enable android connectivity

The randomly generated MAC address will be of the type "Locally Administrated"
https://en.wikipedia.org/wiki/MAC_address#Universal_vs._local_(U/L_bit)

change the radxa scripts to specify a specific (Universally managed) MAC addresses to allow android connection. You can use the one in the script below.

`sudo vi /usr/sbin/radxa-otgutils

search for the configfs_make function and add some lines

```
configfs_make()
{
    if [[ ! -e "/sys/kernel/config/usb_gadget/$UDC" ]];
    then
        modprobe libcomposite
        mkdir -p "/sys/kernel/config/usb_gadget/$UDC"
        pushd "/sys/kernel/config/usb_gadget/$UDC"

        echo "0x1d6b" > "idVendor"  # Linux Foundation
        echo "0x0104" > "idProduct" # Multifunction Composite Gadget
        echo "0x0100" > "bcdDevice" # v1.0.0
        echo "0x0200" > "bcdUSB"    # USB 2.0
        echo "0xEF" > "bDeviceClass"
        echo "0x02" > "bDeviceSubClass"
        echo "0x01" > "bDeviceProtocol"

        popd
    fi
    if [[ ! -e "/sys/kernel/config/usb_gadget/$UDC/strings/0x409" ]];
    then
        mkdir -p "/sys/kernel/config/usb_gadget/$UDC/strings/0x409"
        pushd "/sys/kernel/config/usb_gadget/$UDC/strings/0x409"

        echo "0123456789ABCDEF" > "serialnumber"
        echo "Radxa" > "manufacturer"
        echo "OTG Utils" > "product"

        popd
    fi
    if [[ ! -e "/sys/kernel/config/usb_gadget/$UDC/configs/r.1" ]];
    then
        mkdir -p "/sys/kernel/config/usb_gadget/$UDC/configs/r.1"
        pushd "/sys/kernel/config/usb_gadget/$UDC/configs/r.1"

        echo "500" > "MaxPower"

        popd
    fi
    if [[ ! -e "/sys/kernel/config/usb_gadget/$UDC/configs/r.1/strings/0x409" ]];
    then
        mkdir -p "/sys/kernel/config/usb_gadget/$UDC/configs/r.1/strings/0x409"
        pushd "/sys/kernel/config/usb_gadget/$UDC/configs/r.1/strings/0x409"

        echo "USB Composite Device" > "configuration"

        popd
    fi
## ADD THESE LINES
    if [[ ! -e "/sys/kernel/config/usb_gadget/$UDC/functions/ecm.usb0" ]];
    then 
        mkdir -p "/sys/kernel/config/usb_gadget/$UDC/functions/ecm.usb0"
        echo "04:7c:16:01:29:bd" > "/sys/kernel/config/usb_gadget/$UDC/functions/ecm.usb0/host_addr"
        echo "04:7c:16:01:24:ed" > "/sys/kernel/config/usb_gadget/$UDC/functions/ecm.usb0/dev_addr"
        ln -s "/sys/kernel/config/usb_gadget/$UDC/functions/ecm.usb0" "/sys/kernel/config/usb_gadget/$UDC/configs/r.1/"

        popd
    fi
###########################################
}


```

reboot to be sure

you should now see a usb0 interface when typing
` ip a
that has the specified (host) MAC address.

### configure the ethernet on usb0

configure the USB0 network device with network manager and give it a fixed ip
![this screenshot](../pics/Pastedimage20250727073101.png)

Change networkmanager configuration to also manage USB gadget devices. This way the "Autoconnect" option will work after reboot
```
sudo cp /usr/lib/udev/rules.d/85-nm-unmanaged.rules /etc/udev/rules.d/85-nm-unmanaged.rules 
sudo nano /etc/udev/rules.d/85-nm-unmanaged.rules 
# change the ENV{DEVTYPE}=="gadget", ENV{NM_UNMANAGED}="1" to ="0"
```

reboot

you should now see a usb0 device with a specific MAC address and a specific IP address when typing 

```
ip a

...
3: usb0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 04:7c:16:01:24:ed brd ff:ff:ff:ff:ff:ff
    inet 192.168.192.1/24 brd 192.168.192.255 scope global noprefixroute usb0
       valid_lft forever preferred_lft forever
```

before configuring the client machine, don't forget to enable ssh to be able to login from another device

`sudo systemctl enable ssh

## configuring a linux client 
### check the USB ethernet device name

when listing all ip devices on you machine you shall notice a new device with a familiar MAC address. use this device
```
ip a
...
10: enp0s20f0u4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 04:7c:16:01:29:bd brd ff:ff:ff:ff:ff:ff
...
```
### configure the ethernet device

`sudo nmtui
see screenshot below
 -> edit a connection -> ADD
 - choose a name (usbnet_radxa in my case)
 - fill-in the device (enp0s20f0u4 in my case) - you do not need to enter the MAC address, it will be added automatically
 - Manually configure the IPv4 section and choose an IP address in the same range as above
 - make sure the "Automatically connect" and "Available to all users" options are checked
 - 
![[Pasted image 20250802124832.png]]

exit the Network Manager config and check if the device is configured:
```
ip a
...
10: enp0s20f0u4: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP group default qlen 1000
    link/ether 04:7c:16:01:29:bd brd ff:ff:ff:ff:ff:ff
    altname enx047c160129bd
    inet 192.168.192.168/24 brd 192.168.192.255 scope global noprefixroute enp0s20f0u4
       valid_lft forever preferred_lft forever
...
```

now you can connect to the radxa device via ethernet/ssh

```
ssh radxa@192.168.192.1
```

## configuring an android client 
tested on Samsung Galaxy Tab S8

### Sufficient power is important
When connecting the 3W/3E devices directly to the tablet with a usb-c male-male cable it might occur that the device does not receive enough power from the tablet.
Use a USB-C OTG adapter/splitter that allows you to connect power and data. This works very well.

![[DSC_1572.jpg]]

### Configure the ethernet device on android
First disconnect from WiFi and Cellular to ensure the routing is working as expected.
You can always connect WiFi and Cellular again AFTER you first connected to the Radxa 3W/3E

A new icon will be showing in the status bar above
![[Screenshot_20250802_132443_One UI Home.jpg]]

Follow the steps in the following screenshots to configure the ethernet device with a fixed IP. You can only configure a static IP when the connection is OFF, so check this and enable the connection again after setting the address

![[Screenshot_20250802_132459_Settings_1.jpg]]
![[Screenshot_20250802_132513_Settings_2.jpg]]
Make sure the Ethernet Device is OFF
![[Screenshot_20250802_132529_Settings_3.jpg]]
Then turn it back ON
![[Screenshot_20250802_132542_Settings_4.jpg]]

Now you can connect using Termux or JuiceSSH or others...
![[Screenshot_20250802_132643_Samsung DeX home.jpg]]
