---
layout: page
title: "How-to: Build a Lightweight Homelab on Ubuntu Server 22.04 LTS"
date: 2024-03-27 00:00:00 -0500
categories: Ubuntu, Homelab
weight: 10
menu: howto
---

> NOTE: This how-to is still in active development and will be updated with significant additions in March and April 2024. It was committed for publication so that certain interested parties could see it in the present condition. Sections marked with TODO are still in early draft stages.

## Introduction

This how-to guide walks you through building an extremely lightweight test lab (or homelab). There are many ways to build and run a test lab, and all have their pros and cons, and some labs are more or less suitable for differet applications. This variation can run very well on an inexpensive spare workstation and easily handle several virtual machines. The how-to was built on a refurbished machine with a 4-core Intel I7 processor, 32 GB of RAM, a PCIe 2.5GbE network card, and 1.5TB of storage drives. For a fuller explanation of the design decisions for this particular setup, see the separate article titled [Designing My Homelab Configuration](/posts/2024-03-27-designing-homelab).

## Unsuitable Uses for this Test Lab Configuration

This test lab configuration is appropriate for a narrow range of environments. It is not appropriate for use on embedded SoC and ARM hardware like the Raspberry Pi, nor is it appropriate on high end hardware like multi-socket systems with 128+GB of RAM and many terabytes of storage across multiple physical devices. Other solutions should be used for environments that are considerably smaller or larger than the targets laid out here.

This test lab requires users to have fairly high privileges on the bare-metal system to create and manage instances. Thus, it is not appropriate for environments where many users may be interacting with the lab server itself. The ideal environment for this solution is a single-user homelab.

This is strictly a test lab configuration, and it should not be considered suitable for production or mission-critical usage. It lacks critical production components like redundancy and backup mechanisms. Such modifications could easily be added, but are well beyond the scope of this how-to.

## Step 0: Required Knowledge, Materials, and Information

This how-to does not require you to have deep technical knowledge of many topics, and the steps can be followed with basic competency. However, a basic understanding of [IPv4 classless inter-domain routing (CIDR)](https://en.wikipedia.org/wiki/Classless_Inter-Domain_Routing#CIDR_notation) notation is necessary to successfully modify the example addresses provided.

This how-to is about building a test lab on bare-metal hardware. To follow this tutorial, you will need the following minimum hardware:

1. A functional computer with an AMD64 compatible multi-core CPU, 16GB of RAM, a supported network card (on-board or expansion card), and around 500GB of bootable storage. The form-factor does not matter, and many desktops, laptops, and rack-mount servers meet or vastly exceed these requirements.

  - You will need to know how to get your selected computer to boot from USB. This can be slightly different on every machine, but often involves pressing the Escape, Delete, or Function keys immediately after turning the machine on. A quick search on the internet for your computer’s model name with the phrase “boot from USB” will usually lead to the correct procedure.

2. A USB flash drive with at least 16GB of storage and the Ubuntu Server 22.04 ISO written to it. Follow a tutorial like [this one](https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#1-overview), which also has links for Windows and Mac OS users.

3. An existing functional network with available IPv4 address space, Internet connectivity through this network, and the ability to plug the test lab server into the network with a physical copper or fiber Ethernet cable. The suggested minimum link speed for this port is 1Gbps over copper Cat6 cable.

> NOTE: This test lab can be built exclusively using IPv6 if you have fully functional IPv6 connectivity. The servers and services referenced in this how-to are fully IPv6 capable at the time of this writing. However, IPv6 connectivity is beyond the scope of this how-to.

The following information will be necessary to complete the installation:

1. The configuration information for your existing functional network, including the subnet address, mask, gateway address, and at least one resolver address.

  - This how-to uses 192.0.2.0 for the existing subnet address, a 24-bit network mask (255.255.255.0), a gateway of 192.0.2.1, and resolvers of 192.0.2.2 and 192.0.2.3. Substitute your existing network configuration information anywhere you encounter these values throughout this document.

2. The available IPv4 addresses on your existing network you will be using for the test lab. A group of 16 addresses will be adequate for most use cases. A range that matches a CIDR boundary is recommended.

  - This how-to uses 192.0.2.16 through 192.0.2.31 for this purpose, which can be referenced using CIDR notation as 192.0.2.16/28, making firewall rules easier. Substitute the available addresses you have chosen anywhere you encounter these addresses throughout this document.

3. Network address information for a second (new) network that will function behind network address translation (NAT). 

  - This how-to uses 198.51.100.0/24 for this network. Substitute your network information for this subnet anywhere you encounter 198.51.100.0/24 addresses in this document.

4. Network address information for a third (new) network that will be isolated from the internet for internal-only communication.

  - This how-to uses 203.0.113.0/24 for this network. Substitute your network information for this subnet anywhere you encounter 203.0.113.0/24 addresses in this document.

5. Your name.

  - This how-to uses the fictitious name “Test User” throughout the document, and you should substitute your name anywhere you encounter the name “Test User” throughout this document.

6. A hostname for your test lab server.

  - This how-to uses the hostname of homelab throughout the document, and you should substitute your chosen hostname anywhere you encounter the hostname homelab throughout this document.

7. The local Linux username and password you intend to login with.

  - This how-to uses the Linux username of testuser throughout the document, which should not be confused with the Launchpad username. You should substitute your chosen Linux username anywhere you encounter the username testuser throughout this document. Common usernames like “admin” or “administrator” are discouraged. The fictitious password of “YourPassword” is used throughout this document and must be changed to a strong password of your choice to be secure.

8. A Launchpad username that has been [setup with SSH public keys](https://help.launchpad.net/YourAccount/CreatingAnSSHKeyPair).

  - This how-to uses the username launchpad-testuser throughout the document, which should not be confused with the Linux username. You should substitute your chosen Launchpad username anywhere you encounter the username launchpad-testuser throughout this document.

> NOTE: This how-to uses IPv4 address space reserved for documentation purposes in RFC 6890, including 192.0.2.0/24, 198.51.100.0/24, and 203.0.113.0/24. These addresses must be changed to appropriate address space to build a functional test lab. Public addressing serves no purpose on the second network that runs behind NAT and the third isolated network, so RFC 1918 addresses in the 10.0.0.0/8, 172.16.0.0/12, and 192.168.0.0/16 ranges are recommended. These are available for private use, and this how-to will walk you through setting up network address translation (NAT) that will function with these ranges. This how-to will work with or without NAT on the existing network, so long as no conflicting networks are chosen.

> NOTE 2: A fairly common home network configuration uses a subnet address of 192.168.1.0, a subnet mask of 24 bits (255.255.255.0), and a gateway of 192.168.1.254. Resolver addresses are often delivered by DHCP, but Cloudflare’s public malware-filtered resolvers of 1.1.1.3 and 1.0.0.3 are a great alternative for static assignments. If you are unaware of how your network is configured at this level, you will need to investigate and learn about your network to use this how-to.

## Part 1: Installing the Ubuntu Server 22.04

There are many detailed [tutorials for installing Ubuntu Server](https://ubuntu.com/tutorials/install-ubuntu-server). Refer to those tutorials if you need to deviate from the standard options presented here for the simplest setup.

> *WARNING:* This procedure will repartition and reformat the storage device on the machine, making any existing data difficult or impossible to recover. Do not follow this procedure on a machine containing data you need to keep.

Follow these steps to install Ubuntu Server 22.04 LTS

1. Insert the USB drive with the Ubuntu Server 22.04 LTS image already loaded onto it into a USB port on the server. Some computers only support bootable USB media from certain USB ports. If your machine does not detect your USB drive on the first attempt, try using a different port.

2. Make sure your machine is connected with an appropriate cable from your desired network interface card to your existing network. One functional connection is required for the best installation experience.

3. Turn on the machine and trigger the boot option to boot from USB. Every system is different, so search the web for your computer’s make and model to find the necessary keystrokes to trigger this boot option.

4. The GRUB boot loader will present a boot menu for a few seconds. Do not press any key other than Enter, or you could trigger a custom boot prompt. Pressing the Enter key will advance the booting process instantly, or you can wait for the timeout to trigger the default boot option. Several boot messages will scroll by quickly, and then the system will pause momentarily while the installation software starts.

5. You should be presented with a screen to choose your language for the install. English is the default language and this how-to is in English, so this how-to presumes you will proceed with English. Select your language and press Enter.

6. Next, you are asked to select your keyboard layout. The defaults are usually acceptable, so use the arrow or Tab keys to select “Done” and then press Enter.

7. On this screen, you select the type of installation you want. The default of “Ubuntu Server” will provide the best experience, so use the arrow or Tab keys to select “Done” and then press Enter.

8. The next screen asks you to configure the network. If your network supports DHCP, a functional configuration will already be shown, but a static configuration is desirable for this test lab environment. Select the interface that shows an active link and press Enter to get access to a menu, then select the “Edit IPv4” option and press Enter again.

9. You will be presented with a popup that asks for the “IPv4 Method,” and “Automatic (DHCP)” will be selected by default. Press Enter to see a list of options, highlight “Manual” and press Enter again to select it.

10. After choosing “Manual” for the “IPv4 Method,” you will have the option to input several values. The appropriate values should look something like this (remember to substitute your own values): 

  - Subnet: 192.0.2.0/24

  - Address: 192.0.2.16

  - Gateway: 192.0.2.1

  - Name servers: 1.1.1.3,1.0.0.3

  - Search domains: Blank

11. Select “Save” and press Enter.

12. After the network configuration is validated and applied, select “Done” at the bottom and press Enter.

13. On the “Configure proxy” screen, no changes are needed, so select “Done” and press Enter.

14. On the “Configure Ubuntu archive mirror” screen, no changes are needed. After the message “This mirror location passed tests” appears, select “Done” and press Enter.

15. On the “Installer update available” screen, there is no need to update, so select “Continue without updating” and press Enter.

16. For most systems, the default options on the “Guided storage configuration” screen is acceptable, so select “Done” and press Enter. If your storage needs custom options, that is beyond the scope of this how-to.

17. The installer will present a suggested disk layout. Choose “Done” and press Enter to accept it, then choose “Continue” and press Enter on the “Confirm destructive action” popup. WARNING: This will destroy pre-existing partitions on the system you are installing this on, making any existing data difficult or impossible to retrieve.

18. On the “Profile setup” screen, enter the following values (remember to substitute your own values):

  - Your name: Test User

  - Your server’s name: homelab

  - Pick a username: testuser

  - Password: YourPassword

  - Confirm your password: YourPassword

19. Select “Done” and press Enter.

20. On the “Upgrade to Ubuntu Pro” screen, the default is to “Skip for now,” which is acceptable for this how-to. Select “Continue” and press Enter.

21. On the “SSH Setup” screen, select the “Install OpenSSH server” option and press Spacebar to check the box. Then select the “Import SSH identity” and change the value from “No” to “from Launchpad” and press Enter. In the “Launchpad Username” field that appears, enter “launchpad-testuser” and press Tab out of the field. Do not enable “Allow password authentication over SSH.” Select “Done” and press Enter.

22. On the “Confirm SSH keys” popup that appears, confirm that the fingerprints displayed are correct for the Launchpad account you are using. Select “Yes” and press Enter. WARNING: These keys are the primary mechanism for remotely accessing the test lab from a workstation, so make sure these are correct.

23. On the “Featured Server Snaps” screen, select “Done” and press Enter. 

24. This will start the actual installation process, which can take anywhere from a few minutes to a few hours, depending on the speed of the machine and Internet connection. For much of the installation, the only feedback you will receive is a single character that appears to be a rotating line. This is normal.

25. When “Reboot Now” appears at the bottom, select it and press Enter.

26. You will be prompted to remove the installation medium (the USB drive) and press Enter again to reboot. It is normal to see error messages at this stage, because the Ubuntu installer attempts to eject the media in case it is an optical drive, but USB drives do not accept this eject command.

After the system has rebooted, the base system is installed and you are ready to proceed to part 2 of this how-to. All future parts can be completed directly from the console using a monitor and keyboard hooked up directly to the server, or remotely from another workstation on the same subnet by using SSH with your SSH keypair.

## Part 2: Installing Necessary Packages for the Test Lab

```
#TODO: Finish this section in detail. Summary:
# Add tools necessary to check that the hardware support for virtualization is active
$ sudo apt -y install cpu-checker
# Run the checker to see if KVM support is enabled
$ kvm-ok
# Install the actual packages. Bridge-utils for networking; libvirt-daemon, qemu, and qemu-kvm for virtualization; virtinst for easy creation of new VM instances
$ sudo apt -y install bridge-utils libvirt-daemon qemu qemu-kvm virtinst
```

## Part 3: Setting Permissions for the Test Lab

```
#TODO: Finish this section in detail. Summary:
# Add the user to the correct groups so they can use virsh to create and control VMs without root privileges
$ sudo usermod -aG kvm $USER
$ sudo usermod -aG libvirt $USER
# Start a new login shell to activate the new group membership
$ exec sudo su -l $USER
```

## Part 4: Configuring the Test Lab Bridged Network

```
#TODO: Finish this section in detail. Summary:
# By default, the installed libvirt +  KVM + qemu install will not have a bridged network created.
# A bridged network, which grants VM instances direct access to the pre-existing network the lab server is connected to, is create by defining a network through an XML file and making the virtualization system aware of it
$ cat <<- “EOF” > /etc/libvirt/qemu/networks/br0.xml
<network>
<name>br0</name>
<forward mode=’bridge’/>
<bridge name=’br0’/>
</network>
EOF
$ sudo virsh net-define /etc/libvirt/qemu/networks/br0.xml
$ sudo virsh net-autostart br0
$ sudo virsh net-start br0
```

## Part 5: Configuring the Test Lab NAT Network

```
#TODO: Finish this section in detail. Confirm whether the apt installer prompts for NAT network values. If not, provide the instructions to modify the default virtual network with `sudo virsh net-edit default` to use the 198.51.100.0/24 range.
```

## Part 6: Configuring the Test Lab Isolated Network

```
#TODO: Finish this section in detail. Summary:
# By default, the installed libvirt +  KVM + qemu install will not have any isolated network created.
# An isolated network can be used to connect VMs to each other without connecting them to the world. It is created like a bridged network, because the VMs are bridged to each other, but without the forwarding mode set.
$ cat <<- “EOF” > /etc/libvirt/qemu/networks/internal.xml
<network>
<name>internal</name>
<bridge name=’internal’/>
</network>
EOF
$ sudo virsh net-define /etc/libvirt/qemu/networks/internal.xml
$ sudo virsh net-autostart internal
$ sudo virsh net-start internal
```

## Part 7: Validating the Configuration

```
#TODO: Finish this section in detail. Summary:
# verify the interfaces exist and are configured with `ip addr` commands
# verify the routes look correct with `ip route` commands
# verify default firewall and NAT rules look correct with `nft list ruleset` commands
# verify that the appropriate networks are running and set to autostart with `virsh net-list --all`
```

## Part 8: Using the Test Lab - Moving on to Other How-to Guides

```
#TODO: Finish this section in detail. Summary:
# Point to general documentation on virtinst
# Point to a subsequent how-to series on building a demo high-availability router setup inside the test lab
```
