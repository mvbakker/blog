---
title: "K8s on Raspberry Pi Cluster - Part 1"
date: "2018-09-23T17:19:57+02:00"
categories: ["posts"]
draft: "false"
slug: k8s-cluster-part-1
author: "Michiel Bakker"
---

## Introduction

About a week ago I started learning about K8s. Soon I learned that there are plenty possibilties to run a cluster yourself online (katacoda, play-with-k8s) but I also learned many people installed it themselves on a Raspberry Pi cluster. Challenge accepted!

Below I'll describe why and how I did this (so far). Please note that I am still learning about K8s as well as Ansible, so it could well be that there are other or better ways to do this. Always do your own research of course and I hope this blogpost is helpful.

## Motivation
After learning about Docker and Docker Swarm, I wanted to see what all the K8s fuzz was about. Over the past months I've often read that Docker Swarm is not production ready, however K8s is. Additionally you see that many parties choose K8s over Docker Swarm for running their containers. As I am currently still learning about K8s, I cannot yet say which one is 'better' and I expect that it really depends on the requirements each time you containerize a system/application. Still, so far I've seen that K8s is pretty neat and looks like a very elegant tool for container orchestration.

Therefore my goal is to pass the Certified Kubernetes Administrator (link 1) exam. This is a hands-on exam where one really needs to show their skills in administrating K8s platforms and implement/configure container deployments. To me the best way to learn that is by creating my own cluster on a set of Raspberry Pis I had laying around anyway. Via this blog series I'll share how I set up my cluster and what did and didn't work for me. Enjoy!

## Hardware setup
In my case I already had all the hardware laying around. After a quick research I learned that Raspberry Pi 2B models are able to run K8s so there I went. My hardware consists of:

*	3 Raspberry Pi 2B
*	an old Sitecom switch
*	4 LAN cables
*	3 microSD card 16GB
*	3 microUSB charging cables (from Anker)
*	1 usb charging dock with five ports (note that a Raspberry Pi needs 2 Amps to function properly, do not go below, this will give you all kind of nasty troubleshooting)

Additionally, bear in mind that in order to set everything up you'll need to have:

*	a microSD card reader
*	a monitor and keyboard
*	computer with internet

Moreover, I would recommend to make sure you have a harddrive with enough space to create some backups of all the microSD cards during the setup. In case something doesn't go as planned you don't have to start all over.

## Cluster preparation
This is not the first time I set up a cluster of Raspberry Pis. This is however the first time I want to use Ansible as a configuration management tool. It is a hassle to install and configure multiple Raspberry Pis manually and a mistake/misstep is created easily. First each Raspberry Pi will be installed and configured in general, in the next section I'll explain how to configure the Raspberry Pis for Ansible and how it should (and shouldn't) be installed.

As a brief architecture, I have three RasPi's of which one will be a K8s master and two will be K8s workers. For now all the RasPi's will need to have the same set-up, this will only change once we start configuring K8s.

As a base image I went with [Raspbian Lite](https://www.raspberrypi.org/downloads/raspbian/). Previously I used DietPi a lot, which I really liked, however I read that most people started off with Raspbian Lite for their K8s cluster, so it seemed like a safe choice. After downloading the Raspbian Lite image (I used version labelled 2018-06-27) write it to the three microSD cards. [Here is a link](https://www.raspberrypi.org/documentation/installation/installing-images/README.md) to the Raspberry Pi documentation explaining how to do it, if you are on a linux machine simply run:

	sudo dd bs=4M if=/path/to/image of=/dev/<microSD>

Note: run 'lsblk' on your linux machine to see the path to your disk, i.e. '/dev/sda'. Do not use the partitions '/dev/sda1', but write it to the drive itself.

After all the microSD cards are ready, insert them into the RasPis and hook the first RasPi on to a monitor, internet and keyboard. Then turn it on, if it boots it's fine, well done!

A fresh image always has the user 'pi' and password 'raspberry', this will have to be changed first! Please run:

	sudo raspi-config

Change the password (first option), then go to network options and change to the hostname to something more convenient (I used: master, worker1, worker2). Finally, in this menu, go to 'Advanced Options' and expand the filesystem (first option). Then finish and reboot the system.

After the Raspi is rebooted, we'll update the system, please run:

command: sudo apt-get update && sudo apt-get -y upgrade

It is not handy to work with the RasPi's in this manner. It would be much easier if you could log on to the small machines remotely. We will do this via SSH, therefore we need two things:

*	enable SSH on the RasPi
*	fix the IP-addresses

The first step is the easiest, just run:

	sudo systemctl enable ssh && sudo systemctl start ssh

The second step can be a little more challenging. In my case I had to configure my router to fix the IP-addresses by logging in and matching the MAC-address of the RasPi's to a choosen IP-address. Additionally edit the file at '/etc/dhcpcd.conf' and uncomment the section that states 'define static profile'. As an example, mine looked like:

	# It is possible to fall back to a static IP if DHCP fails:
	# define static profile
	profile static_eth0
	static ip_address=<fixed-IP-address-RasPi>
	static routers=<IP-address-Router>
	static domain_name_servers=<IP-address-Router>

Check if it worked by running the following command after rebooting:

	ifconfig

Now repeat this process for all the RasPi's.

## Ansible installation
Now let's start with the Ansible installation. This will make sure that we won't have to manually configure each RasPi individually anymore, which by now I think you definitely see the advantage. Ansible works via the SSH, therefore we'll need to make sure that this works flawelessly without the need for using passwords. I am not sure yet if this is overkill or necessary, however, I configured it such that each RasPi can SSH without a password request to all the other RasPi's. But first it is a good practice to create a generic user for Ansible, I named her 'ans'. To do this, please run:

	sudo adduser ans

Keep everything generic (password wise). Moreover, 'ans' will need to have the right to sudo without a password. To add 'ans' to the group of sudoers, please run:

	sudo adduser ans sudo

Finally, copy the file '/etc/sudoers.d/010_pi-nopasswd' to '/etc/sudoers.d/010_ans-nopasswd', by running:

	sudo cp /etc/sudoers.d/010_pi-nopasswd /etc/sudoers.d/010_and-nopasswd

This new file needs to be slightly adjusted. Open it using your favorite text-editor (nano, vim, vi, emacs) and change the name 'pi' to 'ans', save and close. Switch to the 'ans' user and test if you can sudo without a password request, run:

	su ans
	sudo apt-get update

Repeat this process for all RasPi's.

Depending on how comfortable you feel to not make a mistake, this would be a good moment to also create a back-up of the images. See [here](https://www.raspberrypi.org/documentation/linux/filesystem/backup.md) how thats done.

It is handy to add the other RasPis to the '/etc/hosts' file on each RasPi. In this way you can just type 'ssh master' for instance, instead of having to type the IP-addresses. It is as simple as adding at the bottom of the file some lines that state:

	# Kubernetes cluster
	192.168.192.10  master
	192.168.192.11  worker1
	192.168.192.12  worker2

Next up, we need Ansible 'ans' user to SSH to each machine without using a password to log in. This is done using ssh-keys. On each machine (also your laptop/desktop which you use to connect to the cluster) run:

	sudo ssh-keygen

Followed by:

	sudo ssh-copy-id <IP-address of other RasPi>

And type the password of 'ans'. Note: the 'ssh-copy-id' command will have to be run from each machine towards all the other machines. Now you should be able to SSH from every machine to another machine without the need to enter a password for 'ans'.

The cluster is now 'Ansible-ready', so let's start installing Ansible. I've tried multiple ways and couldn't find a clear instruction online. Seems like most people did this in 2016 and many just compiled Ansible from the source code. The following options did not work for me:

*	using sudo apt-get install ansible (generic method for Debian, also didn't work after installing dependencies)
*	using sudo pip install ansible (also didn't work after installing dependencies like setup-tools)

What did work was the following:

	sudo apt-get install python3-pip
	sudo pip3 install ansible

Hopefully this works for you as well and it saves you the hassle of trying different approaches and prevents you from having to compile from the source code. Please install it in which ever way you find most convenient.

Additionally, I choose to run Ansible from my desktop, pushing commands towards the cluster from there. Therefore I installed Ansible also locally on my Desktop. This will function as my 'Control Center', but keep in mind that Ansible is able to push configurations from each machine to all the other machines if the Ansible-config files are shared correctly. As I have no idea what kind of system you are running, please look at the Ansible documentation for [the installation instructions](https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).

Now that Ansible is installed everywhere, and Ansible user 'ans' can connect to each system without a password where she has sudo rights without a password, we are ready to create/adjust the Ansible hosts file. This file is generally found at '/etc/ansible/hosts'. Simply add the following to the '/etc/ansible/hosts' file on your 'Control Center' machine, (note: change to IP-addresses if you did not adjust the local /etc/hosts file on the machine):

	[K8s]
	master
	worker1
	worker2

	[master]
	master

	[workers]
	worker1
	worker2

Finally, to check if everything works, run your first Ansible command:

	ansible all -m ping

You should get something similar like this:

	192.168.192.11 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}
	192.168.192.10 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}
	192.168.192.12 | SUCCESS => {
	    "changed": false,
	    "ping": "pong"
	}

If you did, congratulations!! You've now got a RasPi cluster with a functioning Ansible installation!

This was it for the first part of the blog on 'K8s RasPi cluster'.
