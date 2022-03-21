# roger-skyline-1

## V.2 VM Part
* Linux OS: Debian (64 bit)
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

To keep the OS up to date, you will use the `apt-get` utility. This will need `sudo` installed first, so the update can only be done as root:
```
su	# Enters root user
apt-get update -y && apt-get upgrade -y
apt-get install sudo vim -y
```
`update` re-synchronizes the package index files from their sources.
`upgrade` installs the newest versions of all packages currently installed on the system from the sources enumerated in `/etc/apt/sources.list`
