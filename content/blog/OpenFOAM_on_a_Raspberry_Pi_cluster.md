---
title: "OpenFOAM on a Raspberry Pi cluster"
date: "2016-09-07"
categories: ["posts"]
draft: false
slug: OpenFOAM-on-RasPi-cluster
author: "Michiel Bakker"
---

[//]: # (General idea/motivation, Hardware, Software)

So a couple of months ago I got this crazy idea. I wanted to learn more about OpenFOAM and the supercomputers they use at my university they use for CFD (computational fluid dynamics) calculations. Just a quick introduction, computational fluid dynamics calculations are performed to calculate fluid (air, water etcetera) flows for basically all kinds of stuff (airplanes, cars, wind turbines etcetera). These calculations are often very computational intensive and therefore they are performed on a cluster of computers (often called a supercomputer, but its basically just multiple computers). Now OpenFOAM is an open-source CFD application that is widely used for flow analysis. After doing a little bit of research I found out that people actually got OpenFOAM compiled and running on a Raspberry Pi. I got excited and crazy ideas about building a cluster of Raspberry Pis and running multicore flow analyses. I knew right away that this was not going to be fast though. So if you're reading this with the intension to build it yourself since RasPi's are cheap and it all sounds fancy with terms as supercomputer and multicore CFD calculations, don't be mistaken, a simple Intel processor is already much faster!

However, I still wanted to do this! Just because it seemed like a nice challenge and a great learning experience. So I got myself five Raspberry Pi 2B with all the needed extras:

* 5x 16GB microSD cards
* 5x microUSB powercables
* a charging station USB-hub (don't cheap out on this, or it will give you headaches later, a Raspberry Pi 2B needs 2Amps preferably)
* 2x a five-port switch (one 6-port switch would've been better but I had these laying around anyway)
* Couple of LAN-cables

I also build a rack in which I could mount them nicely out of some metal nuts and bolts and some perspex.

[//]: # (PLAATJE VAN CLUSTER)

So in this blog post I'd like to show you what and how I did. Unfortunately I can already say that in the end I did not succeed completely. I was able to run OpenFOAM and I set up the complete system on multiple Pi's. I did manage to run OpenFOAM simulations using four cores on one Raspberry Pi as well, just in the end I did not get it to work using multiple Raspberry Pi. This was mostly because OpenFOAM is a complex application and I couldn't configure it such that it would be able to communicate with the other Pi's and share all the data nicely. I still do not know what is going wrong specifically but I'll share my hypotheses at the end of this post.

Install OpenFOAM on a Raspberry Pi
------------
[//]: # ( Raspbian installation / required modules installation / OpenFOAM installation )

Assuming the reader is already a little bit familiar with a Raspberry Pi and Linux in general. So please install the most recent version of Raspbian, just get it from their website. We'll just be using the terminal, so skip the whole GUI with a desktop environment. I found out (the hard way) that choosing a keyboard different than the regular 'US' will almost always give you trouble. (Honestly, I don't understand why each country needs it's own different keyboard configuration. But I guess that's just me).
It's always a good idea to change your repository mirror server to one close by. Go to the [repository mirror website][mirrorsite] and see which mirror server is the nearest to your home. Change the sources.list by typing in the terminal,

[mirrorsite]: http://www.raspbian.org/RaspbianMirrors

```terminal
sudo nano /etc/apt/sources.list
```

This is the location where the 'apt-get' (your package manager) keeps it sources located. Typ a '#' in front of the first line (this will make the line a comment) and type your repository mirror server below, for instance like

```terminal
deb http://raspbian.mirror.triple-it.nl/raspbian/ wheezy main firmware
```

Which was best for my home. Make sure you now update and upgrade your system.

```terminal
sudo apt-get update && sudo apt-get upgrade && sudo apt-get clean
```

This might take a couple of minutes, especially if it's the first time you do this after installation. It's always best to reboot your system quickly after this before you continue,

```terminal
sudo reboot
```

So lets start the installation of OpenFOAM! Create a directory for the installation.

```terminal
mkdir /home/pi/OpenFOAM
```

Now since we run this all in the terminal without a graphical interface going to a website and downloading the sourcecode of OpenFOAM would be a bit difficult. Or so I thought! But then I found Lynx, a textbased web browser! This works quite funny and its maybe best if you first go to the website of OpenFOAM with a normal web browser on a different computer just to see what you're looking for. However, lynx is quite awesome, please install it,

```terminal
sudo apt-get install lynx
```

Have a look at the manual, although it works quite easily. Use the up and down arrows to go through clickable links, the back arrow for previous page and enter to 'click' on the selected link. Once you've quickly read the manual of lynx (terminal: 'man lynx') enter in the terminal,

```terminal
lynx http://www.openfoam.org/download/source.php
```

And download the files (don't forget to type 'Y' once you've downloaded them, since lynx will ask you whether you'd like to save the downloaded files. When you do not do this, it will not save the downloaded files and you'll have to download them again),

* OpenFOAM-2.4.0.tgz
* ThirdParty-2.4.0.tgz

When I tried to compile OpenFOAM 3 it gave a building error and unfortunately changing a sourcecode so it would compile correctly is still beyond of my skill set.

Unpack the files by running,

```terminal
tar -xvzf OpenFOAM-2.4.0.tgz && tar -xvzf ThirdParty-2.4.0.tgz
```

(If you don't know what the '-xvzf' does, I encourage you to read the manual). (Sorry, not trying to be a bitch, it's just that reading manuals is something I really had to get used to myself). Alright, now in order to compile OpenFOAM from the sourcecode we'll need to do a slight adjustment to two files, namely cOpt and c++Opt which are located at,

```terminal
cd /home/pi/OpenFOAM/OpenFOAM-2.4.0/wmake/rules/linuxARM7gcc
```

Now open 'cOpt' and 'c++Opt' in your favorite editor (or just use nano) and comment the line which ends with '-mfloat-abi=softfp' thereafter uncomment the line that ends with '-mfloat-abi=hard'. Do this for both files and safe them. Then go back to the OpenFOAM-2.4.0 directory,

```terminal
cd /home/pi/OpenFOAM/OpenFOAM-2.4.0
```

During the installation of OpenFOAM the program needs to know stuff about your system. Therefore the global variables need to be given/loaded. You do this by running,

```terminal
source /home/pi/OpenFOAM/OpenFOAM-2.4.0/etc/bashrc
```

Now we can finally compile OpenFOAM,

```terminal
./Allwmake
```

This will take hours/days, since it will only use one core. Once it's done go play with it. I found this amazing [OpenFOAM tutorial][OF-tut] online made by a fellow student. If you don't have any experience with the 'C' or 'C++' language, you may first want to do a tutorial on that. After getting to know OpenFOAM continue with the next section of this post.

[OF-tut]: http://the-foam-house5.webnode.es/products-/

Configure multiple RasPi's for Cluster
------------
[//]: # (comment)

The only thing left is basically setting up each individual RasPi and set up an Network File System (NFS) and distribute public keys so that they can talk freely to one another. Also the network settings per RasPi will have to be configured so that they'll have a static IP address. This is necessary for them to be able to find each other but it's also very handy that you'll be able to SSH into each RasPi for some changes when necessary. These are all basics widely covered on other websites as well.

Start by checking the current configuration. The setup file is called 'interfaces' and is located in at '/etc/network/interfaces'. Use the 'cat' or 'less' command to read it quickly,

```terminal
less /etc/networks/interfaces
```

If you didn't touch this file yet it probably looks something like,

```
auto lo

iface lo inet loopback

auto eth0
allow-hotplug eth0
iface eth0 inet manual
```

etcetera.

This file is used to configure your network settings. In order to set up a static IP address, we first need to know some information about the network we're on. For instance the router address etc. The 'ifconfig' command will give you quite some info. Type,

```terminal
ifconfig
```

and make a note of the current 'inet addr', 'Bcast' and 'Mask' next to the 'eth0'. For me those current values are: (However those could be very different to you!)

* inet addr: 192.168.1.26
* Bcast: 192.168.1.255
* Mask: 255.255.255.0

Another command we need to get the final information is 'route -n'. Go ahead and type it in the terminal,

```terminal
route -n
```

And make a note of the 'Destination' and 'Gateway' values, for me those are:

* Destination: 192.168.1.0
* Gateway: 192.168.1.1

Remember that my values can be very different to your values. Now we'll need to configure the file we checked first. We'll need to set the addresses we just found and set a static IP so we'll always know the address of our RasPi. So in order to configure the RasPi type in the terminal,

```terminal
sudo nano /etc/network/interfaces
```

The configuration file should now open (possibly after you enter the root password). Change this file to,

```
auto lo
iface lo inet loopback

iface eth0 inet static
address "your inet addr"
netmask "your mask"
network "your destination"
broadcast "your bcast"
gateway "your gateway"

auto wlan0
```

and leave everything below as it is. Note that between you should replace for instance '' by your value found at 'Mask' earlier.
Now make sure you DOUBLE CHECK everything before entering the command: ctrl+x
Followed by a 'y' command followed by an enter to make sure you safe the file. Now remove all the old DHCP files by typing in the terminal,

```terminal
sudo rm /var/lib/dhcp/*
```

Now reboot your system by,

```terminal
sudo reboot
```

Wait for your system to reboot and run,

```terminal
ifconfig
```

To check if the new configuration worked. The 'inet addr' should now be the address you have supplied in the '/etc/network/interfaces' file and should now be static. You can now easily SSH to your RasPi by typing in the terminal of your laptop/desktop computer:

```
ssh "your user name on the pi"@"your inet addr"
```

Oke now let's set up SSH, with public/private keys, config-file and all that. This makes accessing each RasPi much easier. Moreover, by exchanging public keys you'll be logged in automatically.

First let's change the hostname of your RasPi, otherwise you'll have all these computers with all the same names by default, namely 'raspberry'. Enter in the terminal of your RasPi:

```
sudo nano /etc/hosts
```

And change the name after the '127.0.1.1' IP address (so do not change the name 'localhost'). Do not change anything else either, safe the file and leave. Now open the hostname file to configure your hostname of the RasPi. Type in the terminal,

```terminal
sudo nano /etc/hostname
```

Remove everything and fill in the same hostname you just entered in the 'hosts' file. Then, in order to initialize the change, run,

```terminal
sudo /etc/init.d/hostname.sh
```

Thereafter reboot your RasPi.

```terminal
sudo reboot
```

Now when you log in again the new hostname should be shown. Now in order to set up a set keys for the SSH, we need to create them first. Run in the terminal,

```terminal
ssh-keygen -t rsa -C "your pi name of choice"
```

Safe the keys in the general '/home/pi/.ssh' directory. Now push your public-key of the computer you wish to login with to the computer you wish to login to. This can be done in the terminal using the following (slightly more advanced to some maybe) command,

```terminal
cat ~/.ssh/id_rsa.pub | ssh "username"@"ip address" 'cat >> .ssh/authorized_keys'
```

Now it will ask for some passwords and thereafter everything should be up and running. Now if you do not want to remember all the static IP addresses of your RasPi's you can add known hosts in the 'hosts' file we edited earlier. Just add the IP address of another and add the hostname of that RasPi. Now when you SSH into it you do not have to type the IP address but you can simply type,

```terminal
ssh "username"@"hostname-of-the-other-Pi"
```

Now unfortunately I could not get it all to work with OpenFOAM. I have tried to run simulations using multiple RasPi's but I kept getting errors. It basically came down to this, they were not able to find the input data which was on the master node. I made a NFS directory from which I ran everything. Also all the environment variables/paths were the same for all the RasPi's so I couln't see where it went wrong and why the slavenodes were not able to get the input data.

There is also not a lot of people playing around with this kinda stuff. I tried solving it, but after a while I just didn't know what to try anymore and also didn't have the time to dive into OpenFOAM. I suspect that I just didn't configure OpenFOAM correctly, which is quite plausible since it's a very complex program. Especially if you want to do stuff  with a computer cluster like this.

My final master thesis project started so I had to let this go sadly enough. So I decided to try and write everything done for you guys and maybe some of you will figure it out. If you do, please let me know, my email address is on the frontpage!
