# roger-skyline-1

## V.2 VM Part
* Linux OS: Debian (64 Bit)
* hypervisor: VirtualBox
* Specified hard-disk size: 8.00 GB VDI (VirtualBox Disk Image) Fixed Size
Once the VM is initialised, go to settings > storage to specify the image of the OS. I used debian-11.2.0-amd64-netinst.iso from https://www.debian.org/distrib/. 

Next, start up the virtual machine to begin setup. To create at least one 4.2 GB partition, pay attention to the `partition disks` stage of installation. Choose `Manual` as the partition method:

I created 2 partitions:
* `primary` mounted on the root `/` of the OS with 4.2 GB capacity
* `logical` partition mounted on `/home` with 4.4GB of space

To check the partition disk spaces, use the `parted` command line utility. To install and use `parted` to show partition sizes:
```
sudo apt install parted
sudo parted
unit GB
print list
```

To keep the OS up to date, you will use the `apt-get` utility as follows:
```
apt-get update -y && apt-get upgrade -y
```
`update` re-synchronizes the package index files from their sources.
`upgrade` installs the newest versions of all packages currently installed on the system from the sources enumerated in `/etc/apt/sources.list`
This will need `sudo` installed first, follow the steps in the section further down.

## V.3 Network and Security Part

### You must create a non-root user to connect to the machine and work
A non-root user was already created during OS set up. To add another user, use the `adduser` utility.

### Use sudo, with this user, to be able to perform operation requiring special rights.
Sudo will need to be installed first, which can only be done as root. Make sure to update the packages to ensure you have the latest version.
```
su	# Enters root user
apt-get update -y && apt-get upgrade -y
apt-get install sudo vim -y
```
exit root user by entering `exit`.

The current user is not yet given sudo permissions however, and needs to be assigned as a sudoer. There are two ways to use this:
* Using the command: `usermod -aG sudo [newuser]`.
* You can also directly modify the `sudoers` file as root, if not `usermod` is not installed and you prefer not to install it.

First, add write permissions to the `/etc/sudoers` file. In the file under `# User priviledges information`, add the new user under the root user with the following:
```
[username] ALL=(ALL:ALL) ALL
```

### We don't want you to use the DHCP service of your machine. You've got to configure it to have a static IP and a Netmask in \30
1. We will configure our Virtual Machine with 2 Adapters. Adapter 1 will be attached to NAT (providing internet access to the virtual machine), Adapter 2 will attach to the host-only adapter which will allow the machines in the host network to communicate with each other. The Adapter 2 attached to the host-only adapter is where we assign the static IP addrress.
 * https://www.codesandnotes.be/2018/10/16/network-of-virtualbox-instances-with-static-ip-addresses-and-internet-access/
 * https://www.quora.com/What-is-the-difference-between-NAT-Bridged-and-Host-Only-networking
 * https://www.nucleiotechnologies.com/what-is-the-difference-between-host-only-nat-bridged-networking-in-virtual-machine-box/

2. VirtualBox configuration
 * We first need to create a host-only network. In the VirtualBox main window, go to Global Tools -> Host Network Manager and click on "create". This should create a network named "vboxnet0", disable the DHCP server for this network. and assign the following configurations for the adapter:
   * IPv4 Address: 192.168.10.42 (This can also be any IP address you prefer)
   * IPv4 Network Mask: 255.255.255.252 (Ending in 252 is the \30 netmask: https://www.freecodecamp.org/news/subnet-cheat-sheet-24-subnet-mask-30-26-27-29-and-other-ip-address-cidr-network-references/)
 * Next the Virtual machine needs to be configured in Settings -> Network, as follows:
   * Adapter 1 - attached to NAT
   * Adapter 2 - attached to Host-only Adapter (this should automatically be assigned to the vboxnet0 host network we just made).

3. Configure static IP address.
  * The VM should be able to access the outside world as normal, but will also need to be configure with the static IP to allow host network communication. Edit teh file /etc/network/interfaces as follows:
```
# This file describes the network interfaces available on your system
# and how to activate them. For more information, see interfaces(5).

source /etc/network/interfaces.d/*

# The loopback network interface
auto lo
iface lo inet loopback

# The primary network interface
auto enp0s3
iface enp0s3 inet dhcp

# The secondary network interface (Host-only adapter)
auto enp0s8
iface enp0s8 inet static
	address 192.168.42.2
	netmask 255.255.255.252
```

 * Restart the network service with `sudo service networking restart`. You can check that you have assigned the static IP using ifconfig
