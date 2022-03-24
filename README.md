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
1. We will configure a bridged adapter connection for the network settings. In the settings -> network, set the adaptor to bridged adapter. The bridged mode replicates another node on the physical network that the host is connected to. The VM will receive its own IP address, in this case, we are going to assign a static IP 10.12.203.42. The VM will be accessible by all computers on the host network.
 * https://linuxconfig.org/how-to-setup-a-static-ip-address-on-debian-linux
 * https://www.quora.com/What-is-the-difference-between-NAT-Bridged-and-Host-Only-networking
 * https://www.nucleiotechnologies.com/what-is-the-difference-between-host-only-nat-bridged-networking-in-virtual-machine-box/
 
2. Edit the `/etc/network/interfaces` file to setup the primary network. Change the primary network line as below:
```
# The primary network interface
auto enp0s3
```

3. Create an enp0s3 file in the directory `/etc/network/interfaces.d/` and add:
```
iface enp0s3 inet static
	address 10.12.203.42
	netmask 255.255.255.252
	gateway 10.12.254.254
```

Note that since the VM is going to be on the same network as the host, assign an address as `10.1X.0.0`. Where X is the cluster. Choose an IP that is not already taken.

 * Restart the network service with `sudo service networking restart`. You can check that you have assigned the static IP using ifconfig

### You have to change the default port of the SSH service by the one of your choice. SSH access HAS TO be done with publickeys. SSH root access SHOULD NOT be allowed directly, but with a user who can be root.
Change the the sshd config file `/etc/ssh/sshd_config`. I changed the line `# Port 22` by uncommenting, and adding a custom number `4242`. Then restart the sshd service with:
```
sudo service sshd restart
```
With this, you can already login via ssh with `ssh caruy@10.12.203.42` -p 4242`.

At this point, you are still loggin in via password. We now need to set up SSH access with public keys instead.

From the terminal, if you don't already have an id_rsa, run `ssh-keygen` to generate a key pair. You can then install the public key on the Virtual Machine OS with the following syntax
```
ssh-copy-id [username]@[server IP address] -p [port number]
```
It will prompt you for a password. After it is successful, you should be able to log in without using a password.

To disable the root login directly, edit the `/etc/ssh/sshd_config` file in the VM, chainging the `PermitRootLogin` setting to `no`

### You have to set the rules of your firewall on your server only with the services used outside the VM.
We will be using Uncomplicated Firewall ([UFW](https://wiki.ubuntu.com/UncomplicatedFirewall)), to set up our firewall. UFW was developed to make `iptables` firewall configureation easier. 

1. Install UFW with `sudo apt-get install ufw`. 
2. Set up default policies to deny all incoming traffic, and allow all outgoing traffic. The default rules handle traffic that do not explicitly match any other rules.
```
sudo ufw default deny incoming
sudo ufw default allow outgoing
```
3. We need to explicitly allow the SSH connection, since the default rule as it is would deny any incoming ssh connection. Since we have configured a custom port for the SSH daemon with `4242`, we need to specify this:
```
sudo ufw allow 4242
```
4. Finally, enable the UFW to activate the firewall.
```
sudo ufw enable
```
For now, we won't enable any other connections such as HTTP on port `80` or HTTPS on port `443`. We might need to add these rules to be enabled in a later part. For more information on setting up the firewall: https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10


