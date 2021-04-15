---
layout: post
title: Homelab! Configuring my Catalyst switches!
---

Hello everyone. I decided to pick up three Cisco Catalyst 2960-C switches to play with for a while. This post is about the basic configurations I made, which include hostnames, setting up the switches for remote management via SSH, and configuring enable and VTY line passwords

----------------------------

### Accessing the switches

![170004534_3798679133556954_8898431508470903884_n](https://user-images.githubusercontent.com/79895144/113953550-ed94b300-97cc-11eb-9d70-1b4dba65b521.jpg)
Here's my little setup. I crimped some cat-5 cables and got myself a console cable to do the initial configuration. 
![1 minicom settings](https://user-images.githubusercontent.com/79895144/113954648-19b13380-97cf-11eb-9b3b-7f5d8c235322.png)
I installed Ubuntu on my spare laptop just to start getting into Linux because that's a great OS to master. At first, I decided to use [minicom](https://help.ubuntu.com/community/Minicom), a serial communication program, to console into the "brand new", unconfigured switches. But later I decided to use PuTTY.
My console settings were: baud 9600, 8 data bits, no parity, and 1 stop bit. 

With a switch that has no previous configurations, we cannot remotely manage it. We need to console into the switch to make the initial configurations that can allow remote management. This is done via a console cable, the one that I have specifically is USB-C --> RJ-45. I will then use a terminal emulation program to terminate the serial connection and get into the switch. 

![sw2 basic config](https://user-images.githubusercontent.com/79895144/113956466-5a5e7c00-97d2-11eb-8ddb-be3a8678eec5.png)
I've consoled into the switch and am now doing some initial configurations. I am prompted on whether I wanted to do the intial configuration dialog, I am not sure if anyone actually does this, I type no. I then do the following commands: 
>**enable** 

This enters me into the enable mode, or privileged EXEC. There are multiple access levels in Cisco switches and routers: user EXEC, privileged EXEC, and global configuration. User exec is the weakest, only allowing you to ping and very limited show commands. Privileged EXEC is a bit more powerful, you are able to make some pretty substantial changes to the device such as reload it and save its running configuration. Global config allows you to make changes to the control plane of the device, like routing protocols and interfaces. There are submodes, such as interface configuration and VLAN configuration, which fall under global config as well. 
>**configure terminal**

This moves me to global config.
>**hostname (Sw2)**

Creates the hostname.
>**enable secret (Teemo)**

This command is telling the switch to ask for a password from anyone who tries to access the enable mode. Cisco password methodology is a mess, there's the enable password, username password, secret, and username secret commands. Using username secret and enable secret are your best bets, since the other passwords show the password in clear text in the running configuration as well as in the command line as it is typed to access enable mode. The "secret" command not only doesn't show your password in clear text as you type it, but uses MD5 hash in the running configuration.
In this case, our password to access the enable mode is Teemo, caps sensitive. 
>**line vty 0 15**

This is accessing our virtual teletype lines, these control our Telnet and SSH lines. Notice that this places us in the (config-line) subconfiguration mode.
>**username (Teemo) secret (Teemo)**

We have created the username and password that users must know in order to access the switch. Think of it like doors. They need to enter the house, or the switch, with a username and password. Then, in order to get into one of the rooms, or the enable mode, they need to supply another password. In our case everything is the same.
>**login local**

This is telling the switch to prompt the user with the locally configured username and password.
>**exit**

We exit the subconfig mode. Sometimes this isn't neccessary as you can jump between different configuration modes without typing exit, but for clarity I typed it here.
>**ip domain-name (Sw2)**

Configures the specific domain name of the device. This is necessary for security certificate generation for SSH, HTTPS, and IPSEC. 
>**vlan (2)**

This creates our virtual LAN (VLAN). We are going to be using VLAN 2 as our management VLAN. In order for us to SSH into the other switches, we need to not only be in the same subnet, but make sure that our frames are being forwarded tagged. Meaning, the Ethernet header needs to have the VLAN ID so the switches do not drop our traffic. Hence the creation of a new VLAN: we can better regulate where the switches are sending our management frames (through trunk links) and improve our ability to troubleshoot. 
>**name MGMT**

The name of our VLAN.
>**ip address (192.168.2.2 255.255.255.0)**

Switches allow a single IP address for remote management. We're going to be using a simple RFC 1918 address with a /24 subnet mask. Note that I configured my laptop with a static IP address in the same subnet.
>**exit**

Exiting again.
>**interface f0/8**

We're entering our fast Ethernet 0/8 port, which will be the port which receives our management traffic. 
>**switchport mode access**

I am manually configuring the port mode. It's an access port, which transmits and receives data from a single VLAN only. 
>**switchport access vlan 2**

We are specifying the VLAN of the port.

Finally, notice that I also had shut down many ports with the interface range and shut commands. This is just good security, as you don't want open and easily accessible ports. 

We have some basic configurations, now let's configure SSH.

![setting up ssh on sw 2](https://user-images.githubusercontent.com/79895144/113960600-90532e80-97d9-11eb-9eb2-749fa411e799.png)

