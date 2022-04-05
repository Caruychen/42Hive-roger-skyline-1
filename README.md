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
sudo ufw allow 80	# Allows http traffic
sudo ufw allow 443	# Allows https traffic
```
4. Finally, enable the UFW to activate the firewall.
```
sudo ufw enable
```
For more information about firewalls:
* https://www.digitalocean.com/community/tutorials/how-to-set-up-a-firewall-with-ufw-on-debian-10
* https://opensource.com/article/18/9/linux-iptables-firewalld

### You have to set a DOS (Denial Of Service Attack) protection on your open ports of your VM.
We will be using `fail2ban` to set-up DOS protection, and the Apache Benchmark tool (ab) to test whether our DOS protection is working. Note that apache2 also needs to be installed on your VM. Apache2 is the webserver software that will serve HTTP requests (https://www.wpbeginner.com/glossary/apache/)

1. Setting up Apache2 web server (https://www.layerstack.com/resources/tutorials/Installing-Apache-server-on-Linux-Cloud-Servers):
	* run `sudo apt-get install apache2`. Then test that the webserver is running by entering the http://[IP Address]/ into your browser. This should give you an Apache2 default webpage,the html for which is located at `/var/www/html/index.html`
2. Create local file sudo nano /etc/fail2ban/jail.d/jail-debian.local
```
  [DEFAULT]
  bantime  = 10m
  findtime  = 10m
  maxretry = 5

  [sshd]
  enabled = true
  port = 50113
  maxretry = 3
  findtime = 300
  bantime = 600
  logpath = %(sshd_log)s
  backend = %(sshd_backend)s

  [http-get-dos]
  enabled = true
  port = http,https
  filter = http-get-dos
  logpath = /var/log/apache2/access.log
  maxretry = 300
  findtime = 300
  bantime = 600
  action = iptables[name=HTTP, port=http, protocol=tcp]
```
3. Create the filter: create file /etc/fail2ban/filter.d/http-get-dos.conf and copy the text below in it:
```
  [Definition]
  failregex = ^<HOST> -.*"GET.*
  ignoreregex =
```
When a line in the service’s log file (`/var/log/apache2/access2.log`) matches the failregex in its filter, the defined action is executed for that service. ignoreregex patterns to filter out what is normal server activity.

4. Restart service by `sudo ufw reload` and `sudo service fail2ban restart` to apply settings
	* command to debug fail2ban: /usr/bin/fail2ban-client -v -v start
5. Activate fail2ban `sudo systemctl enable fail2ban`
6. Check status of fail2ban: sudo systemctl status fail2ban
	* You can an also see the rules added by Fail2Ban by running the following command: sudo iptables -L
7. Tested with slowloris using second virtual machine, with 150 sockets. After 300 attempts in 5m, the second VM's IP address is banned for 10 minutes.
8. The fail2ban response to requests can be found in log file: `tail -f /var/log/fail2ban.log` And by checking all of the banned http actions `sudo fail2ban-client status http-get-dos`

### You have to set a protection against scans on your VM’s open ports.
Using Portsentry can be complementary to fail2ban. Portsentry blocks IP addresses that are aiming to identify open ports on the server: https://en-wiki.ikoula.com/en/To_protect_against_the_scan_of_ports_with_portsentry
1. Install Portsentry: sudo apt-get update -y && sudo apt-get install portsentry. Then stop the service while we configure porsentry `sudo service stop portsentry`.

2. We will use portsentry in advanced mode for the TCP and UDP protocols, edit the `/etc/default/portsentry` file:
```
TCP_MODE="atcp"
UDP_MODE="audp"
```

3. We need to explicitly activate blocking with portsentry, so edit the section `Ignore Options` in `/etc/portsentry/portsentry.conf` such that:
```
BLOCK_UDP="1"
BLOCK_TCP="1"
```
4. We will configure portsentry to drop routes using iptables. Incoming request packets from malicious IP addresses will be dropped using the `iptables` command with `DROP`. We are using linux, so will uncomment the line:
```
KILL_ROUTE="/sbin/iptables -I INPUT -s $TARGET$ -j DROP"
```
5. the above line is the packet filter, which `portsentry.conf` mentions as the PREFERRED method, and we needed to explicitly uncomment it. The default method uses the route ccommand, and is supposedly the leaast optimal way of blocking and does not provide complete protection against UDP attacks. We can comment out the following line:
```
#KILL_ROUTE="/sbin/route add -host $TARGET$ reject"
```

6. restart the service and check its status:
```
sudo service portsentry start
sudo service portsentry status
```

7. You can check open ports and which application is listening on what port by using `lsof -i`. Portsentry logs in the file `/var/log/syslog`, where you can see logs of any attacks. The following command also lists blocked IP addresses via iptables `sudo iptables -L -n -v`. 

8. You can also test that portsentry has detected a port scan, by runningg `sudo nmap -v -A -sV 10.12.203.42` from another VM. the `/var/log/syslog` should who an `attackalert` from the attacking host IP, and subsequently dropping the packets from the attacker IP.

### Stop the services you don’t need for this project.
1. We need to first discover every available unit file within the `systemd` paths, and filter for active service units that are enabled: `sudo systemctl list-unit-files --type=service | grep "enabled "`
	* We include sudo as the state of the operating machine may be different if you are logged in as non-root user.
	* `list-unit-files` shows all installed unit files, instead of units currently in memory as in `list units`
	* `--type=service` makes sure we list active service unit types.
	* We then grep for any units that are "enabled ". We leave a space, because we want to only check against the "STATE", otherwise we will also see those enabled from vendor preset which is not reflective of current state.

2. We can then stop, and disable the services we do not need. Disabling the service will prevent it from starting automatically whenever the machine is launched.
* `sudo systemctl stop [application].service`
* `sudo systemctl disable [application].service`

3. Below are the services we need for the project:
```
apache2.service				- Apache Hypertext Transfer Protocol (HTTP) Server
apparmor.service			- kernel enhancement to confine programs to a limited set of resources
cron.service				- daemon to execute scheduled commands (Vixie Cron)
fail2ban.service			- a set of server and client programs to limit brute force authentication attempts
getty@.service				- opens a tty port, prompts for login and invokes /bin/login command.
networking.service			- raises or downs the network interfaces configured in /etc/network/interfaces
rsyslog.service				- syslog server, for managing logs
ssh.service					- OpenSSH remote login client
systemd-timesyncd.service	- used to synchronize the local system clock with a remote network time protocol server.
ufw.service					- for managing a netfilter firewall
```

### Create a script that updates all the sources of package, then your packages and which logs the whole in a file named /var/log/update_script.log. Create a scheduled task for this script once a week at 4AM and every time the machine reboots.
1. Create a script at `/usr/local/bin/update_pkgs
```
#!/bin/bash
sudo apt-get update -y >> /var/log/update_script.log
sudo apt-get upgrade -y >> /var/log/update_script.log
```
We store the script in `/usr/local/bin` as it is commonly for programs that a normal user can run: https://unix.stackexchange.com/questions/4186/what-is-usr-local-bin

2. The file needs to be given execute permissions so: `sudo chmod +x /usr/local/bin/update_pkgs`.

3. Update crontab file with `sudo crontab -e`, and add:
```
0 4 * * 0 sudo /usr/local/bin/update_pkgs
@reboot sudo /usr/local/bin/update_pkgs
```

### Make a script to monitor changes of the /etc/crontab file and sends an email to root if it has been modified. Create a scheduled script task every day at midnight.
1. We will need to install the `mailx` command line tool, run `sudo apt-get install mailutils`
2. Create a script `/usr/local/bin/monitor_cron` with relevant permissions
```
#!/bin/bash
if [ ! -e /etc/crontab.back ]
then
	exit 0
fi
DIFF=$(diff /etc/crontab /etc/crontab.back)
sudo bash -c "cat /etc/crontab > /etc/crontab.back"
if [ "$DIFF" != "" ]
then
	echo "crontab monitor: change detected, alerting root!"
	echo "$DIFF" | mailx -s "crontab monitor: change detected" root
fi
```
3. Add a task to the crontab `0 0 * * * sudo /usr/local/bin/monitor_cron`
4. To read the mail, execut `mailx`

### Web Part
#### You have to set a web server who should BE available on the VM’s IP or an host (init.login.com for exemple). About the packages of your web server, you can choose between Nginx and Apache. You have to set a self-signed SSL on all of your services.
We're using the wordle Rush project which was made in javascript for this part. It's a wordle game assistant.

1. We will first create the SSL certificate using the `openssl` command:
```
sudo openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/ssl/private/apache-selfsigned.key -out /etc/ssl/certs/apache-selfsigned.crt
```
* openssl: The command line tool for creating and managing OpenSSL certificates, keys and other files.
* req: subcommand that specifies we want to use X.509 certificat signing request management.
* -x509: modifies the previous subcommand by telling the utility that we want to make a self-signed certificate.
* -nodes: tells OpenSSL to skip the option to secure our certificate with a passphrase.
* -days 365: specifies length of time that the certificate will be considered valid.
* -newkey rsa:2048: Specifies that we want to generate a new certificate and a new key at the same time. The `rsa:2048` portion tells it to make an RSA key that is 2048 bits long.
* -keyout: tells OpenSSL where to place the generated private key.
* -out: tells OpenSSL where to place the certificate.

2. Create an Apache configuration snippet with strong encryption settings. We will set up a strong SSL cipher suite, with a snippet in the `/etc/apache2/conf-available` directory:
```
SSLCipherSuite EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH
SSLProtocol All -SSLv2 -SSLv3 -TLSv1 -TLSv1.1
SSLHonorCipherOrder On
# Disable preloading HSTS for now.  You can use the commented out header line that includes
# the "preload" directive if you understand the implications.
# Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
Header always set X-Frame-Options DENY
Header always set X-Content-Type-Options nosniff
# Requires Apache >= 2.4
SSLCompression off
SSLUseStapling on
SSLStaplingCache "shmcb:logs/stapling-cache(150000)"
# Requires Apache >= 2.4.11
SSLSessionTickets Off
```
The `Strict-Transport-Security` header (HSTS) is disabled. While it provides increased security, there are far reaching implications, and complexities later on: https://hstspreload.appspot.com/

3. Modify the default Apache SSL virtual host file `/etc/apache2/sites-available/default-ssl.conf`. First, we will make a backup of this file before changing it.
```
sudo cp /etc/apache2/sites-available/default-ssl.conf /etc/apache2/sites-available/default-ssl.conf.bak
```
We will need to modify the following lines in the file as below:
```
ServerAdmin [you_email@example.com]
ServerName [server_domain_or_IP]
SSLCertificateFile [/etc/ssl/certs/apache-selfsigned.crt
SSLCertificateKeyFile [/etc/ssl/private/apache-selfsigned.key]
```

4. Modify the HTTP Host file to redirect to HTTPS. Since the server provides both unencrypted HTTP and encrypted HTTPS traffic, we can improve security by redirecting HTTP to HTTPS automatically. Modify the `/etc/apache2/sites-available/000-default.conf` file by adding a `Redirect` directive:
```
Redirect "/" "https://[your_domain_or_IP/"
```

5. Enable changes in Apache. We can enable the SSL and headers modules in apache, enable SSL-ready Virtual Host, and then restart apache to put these changes into effect.
	* Enable `mod_ssl` (Apache SSL module) and `mod_headers` which is needed by some of hte settings in the SSL snippet above, with the `a2enmod` command (enables or disables apache2 modules):
	```
	sudo a2enmod ssl
	sudo a2enmod headers
	```
	* enable SSL Virtual Host with a2ensite command:
	```
	sudo a2ensite default-ssl
	```
	* To check to make sure there are no syntax errors in our files:
	```
	sudo apache2ctl configtest
	```
	* As long as you get `Syntax OK`, then you should be fine. To activate the new configs, run:
	```
	sudo systemctl reload apache2
	```

6. Testing encryption. You can test the encryption by entering `https://` followed by the domain or IP. You should get an error message in the browser stating `Your connection is not private`. This is to be expected, since we have only set up encryption. We don't yet have third party validation of our hosts's authenticity, which would require the browser to add us to the trusted certificate authorities.  You can continue to proceed to the page, and will see the URL tab indicating `unsecure`. The connection is still encrypted however.

7. You can change the Redirect to permanent, by modifying the following line in: `/etc/apache2/sites-available/000-default.conf`
```
Redirect permanent "/" "https://[domain_or_IP]/"
```

Instructions about setting up SSL learned from: https://www.digitalocean.com/community/tutorials/how-to-create-a-self-signed-ssl-certificate-for-apache-in-debian-10

### Propose a functional solution for deployment automation.
We set up a deployment script that will automatically patch updates to the contents in `/var/www/html/`. It will do so by running `diff` against the contents inside the relevant development directory, in this case `~/42Hive-wordle-assistant/assistant` with `/var/www/html`, and using the patch command to deploy updates:
```
#!/bin/bash
DIFF=$(diff -ru ~/42Hive-wordle_assistant/assistant /var/www/html)
if [ "$DIFF" != "" ];
then
	sudo sh -c 'diff -ru /var/www/html /home/caruy/42Hive-wordle_assistant/assistant > /var/www/html/web.patch'
	sudo patch -d/var/www/html/ < /var/www/html/web.patch
	sudo rm /var/www/html/web.patch
else
	echo "No changes detected"
fi
```
