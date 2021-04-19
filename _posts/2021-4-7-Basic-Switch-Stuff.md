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

### Basic config

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

![show run of ssh password](https://user-images.githubusercontent.com/79895144/115182843-8d84f300-a08f-11eb-8ffd-26aed28c56a8.png)

Here's a quick screenshot of the running config showing our passwords (This is on SW3, but the configuration is the same for all my switches) You can see the line `enable secret 5 (gibberish)` and `username Teemo secret 5 (gibberish)`. The "5" value is a code meaning the password was encrypted using MD5. If you scroll down on the running-config (not pictured), you will find the VTY lines 0 15, with the `login local` command we configured earlier. 

This is telling the switch to prompt the user with the locally configured username and password.
>**exit**

We exit the subconfig mode. Sometimes this isn't necessary as you can jump between different configuration modes without typing exit, but for clarity I typed it here.
>**ip domain-name (Sw2)**

Configures the specific domain name of the device. This is necessary for security certificate generation for SSH, HTTPS, and IPSEC. 
>**vlan (2)**

This creates our virtual LAN (VLAN). We are going to be using VLAN 2 as our management VLAN. In order for us to SSH into the other switches, we need to not only be in the same subnet, but make sure that our frames are being forwarded tagged. Meaning, the Ethernet header needs to have the VLAN ID so the switches do not drop our traffic. We also just don't want all of our traffic in the default VLAN, both for security purposes and because I plan to add more VLAN's later on.  
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

In all three of our switches, SW1, SW2, and SW3, I created the management VLAN, VLAN 2. I have also added an IP address in that VLAN interface for remote management. Additionally, I configured the access ports of the VLAN's and created an enable password on each switch. The management IP's for my switches are:
- SW1: 192.168.2.1/24
- SW2: 192.168.2.2/24
- SW3: 192.168.2.3/24

### Configuring Secure Shell version 2

We have some basic configurations, now let's configure SSH.

![setting up ssh on sw 2](https://user-images.githubusercontent.com/79895144/113960600-90532e80-97d9-11eb-9eb2-749fa411e799.png)

>**crypto key gen rsa**

This will generate our SSH key. Aftwewards, we will configure an RSA key size larger than 768 bits for SSH v2 by typing `768`, the more secure and widely used version of Secure Shell. We follow up with `ip ssh v 2` to enable version 2.

![screenshot 1](https://user-images.githubusercontent.com/79895144/115182318-a04af800-a08e-11eb-9e4c-55d14268efc8.jpg)


So we have our SSH keys on each switch. Now we need to make a trunk port for the switches to carry our management traffic. Why? Because of VLAN tagging. Switches have two port modes- access and trunk. As we've discussed, access ports are for a single VLAN, only processing frames with the correct VLAN ID in the frame header. What if you have a switch with 10 VLAN's, with many access ports corresponding to these VLAN's? This is where trunk links come in. A trunk link is telling the switch, "process and forward any frames with this VLAN ID", and unrecognized frames will be sent to the native VLAN. For example, if you are in VLAN 10, and you send a frame intended for a buddy a few switches away, but the switches trunk ports are only configured to allow VLAN 20 and 30, your frame will be sent to the native VLAN and your buddy will never receive your traffic. 

>**switchport mode trunk**

We're defining the port mode. Note that I went `into interface f0/1` prior to this command, so we're in interface configuration mode. Also take note that some gigabit switches may require you to enter `switchport encapsulation dot1q` prior to this command to enable 802.1Q tagging. This is because Cisco offers their proprietary Cisco ISL, another VLAN technology.  

>**sw trunk allowed vlan 2**

We're defining the VLAN's allowed on the trunk link. In our case, we only have two VLAN's, our management VLAN for remote management traffic, and our native VLAN for everything else. It may seem a little unnecessary now, but it is critical to seperate your traffic for security. In the future, when I add more VLAN's, I will simply add the VLAN's to the existing trunk port. Please, for the love of God,  to ADD VLAN's to the link, use the commmand `switchport trunk allowed vlan add (VLAN ID's)`. 

Also notice I applied the `copy run start` command. This is because any changes you make to the switch will apply only to the running-config of the switch, which is hosted on the RAM, which is volatile and erased upon the switch rebooting. The startup-config, hosted on the NVRAM, is not erased after reboot. So with `copy run start` we are copying our running-config to our startup-config, essentially saving our changes. Which is what I'm doing since I'm not leaving my switches plugged in doing nothing. One more thing, notice I put `do` in front of `copy run start`. This is because show commands and changes to the device functions like saving your running-config and reloading are actually USER EXEC commands, not CONF-T commands. But adding `do` to the front of show commands and the like will allow you to show tables, save the running config, etc, without jumping back and forth between CONF-T and USER EXEC. 

Now that we have created our trunk links on our switches, created our passwords, created our SSH keys, created our access ports, we're now ready to SSH into the switches. At this point I remove my console cable and plugged in another Cat5. 

### Testing our setup

![logging into sw 2](https://user-images.githubusercontent.com/79895144/115183382-81e5fc00-a090-11eb-837a-a81eb0056119.png)

Success! You can see at the top I was prompted to log in with a username and password. This is the `line vty 0 15` and `login local` commands at work, along with our `username (Teemo) secret (Teemo)` commands. Next, I was prompted after typing `en` to enter USER EXEC mode. Afer typing that password, I can now access the switch. Still plugged into SW2, let's see if I can access SW1 and SW3. This will test if we have configured our trunks correctly. 

![screenshot 1](https://user-images.githubusercontent.com/79895144/115183718-36801d80-a091-11eb-8f8a-1d857738f2f2.jpg)

This has worked as well. To explain it better, I'll draw up a quick diagram of our little network. 

![screenshot 2](https://user-images.githubusercontent.com/79895144/115184186-2288eb80-a092-11eb-97c8-873e47c99852.jpg)

My laptop's copper connection is in SW1's F0/8 access port. When I SSH into SW1, the switch will realize the frames I am sending are intended for them. (This is the work of ARP, we can cover this in a future post) However if I want to SSH into SW2 or SW3, our frame will first enter the F0/8 access port. Since we configured it with VLAN 2, the switch will tag our traffic with a VID of 2. It will then forward our frames out the trunk link on F0/2- this works because we allowed VLAN 2. This extends from SW1 to SW2, and SW2 to SW3. With this setup, in a larger deployment I would be able to SSH into any switch I wanted by simply having access to the network. This remote management is much better than walking to the wiring closet and consoling in. It also allows us to have multiple SSH sessions at once, so we can make changes to the switches much faster.

------------------------

### Final remarks

We have done a pretty basic setup on three physical switches. There's a lot more to elaborate on, but for now I'll work on a detailed **Address Resolution Protocol** (**ARP**), explanation as well as STP. Thank you for reading! 
