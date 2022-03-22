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
1. We will use the `bridged adapter` for the VM's network settings. Go to VirtualBox settings > Network, and change the enabled network adapter to `bridged adapter`.
 * The `bridged adapter` replicates another node on the physical network, so it ccan have an IP address on the same domain as the host. It will behave just like it was another machine on the same network as the host. 
 * https://www.quora.com/What-is-the-difference-between-NAT-Bridged-and-Host-Only-networking
 * https://www.nucleiotechnologies.com/what-is-the-difference-between-host-only-nat-bridged-networking-in-virtual-machine-box/
