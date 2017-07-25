# Juniper vRR on PVE

Tutorial for installing the Juniper vRR on Proxmox VE

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See deployment for notes on how to deploy the project on a live system.

### Prerequisites

What things you need to install the software and how to install them

```
Server has installed Proxmox VE 4.4
	 8GB RAM, CPU Intel E5-2620 and two NICs
jinstall64-vrr-14.2R7.5-domestic.img images
```

## Installing

### Part1: Proxmox VE 4.4 installation and preparation
Download the installation .iso file from Proxmox downloads and create a bootable USB stick. For NVMe boot support Proxmox needs to be at least version 4.2-725d76f0-28 which was released on the 27th April 2016.

Set the BIOS of your PC to boot from the USB key and proceed to boot.

When prompted, select Install Proxmox VE
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-1.png "")

Accept the EULA agreement by clicking Next.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-2.png "")

Select the hard disk you wish to install to and click Next. Here I am installing to my Intel 750 NVMe PCI drive. Booting from NVMe devices is a recent addition to Proxmox and I.ll go over whats required to enable this later on in this guide. Feel free to leave the options as default.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-3.png "")

Set Country, timezone and keyboard layout then click Next
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-4.png "")

Enter a strong password and an email address. Its worth using a valid email address as this can be used to reset your password in future should you need to for any reason.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-5.png "")

Setup the network as per your local requirements, here I am connecting to my VL20_VPNLAN with a fixed address in the range I use for critical production servers.
```
Hostname = proxmox.local.lan
IP Address = 192.168.20.12
Netmask = 255.255.255.0
Gateway = 192.168.20.1
DNS = 192.168.20.1
```

Click next to proceed.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-6.png "")

Reboot and remove CD/USB when prompted and click Next to restart.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-7.png "")

If everything has worked correctly you will see a screen that looks something like this with lines of text scolling up the screen.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-8.png "")

Once the boot sequence has completed, you.ll be presented with a login screen. Proxmox has now completed booting and we can commence with our configuration.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-9.png "")

We aren.t going to jump into the web GUI immediately, first we will log into the command shell and setup the underlying Linux systems for a some key services we will need for reliable operation of Proxmox.

Login to console with username **root** and the password you used during the initial install.

### Part2: Update Linux packages for non-licenced use.

The pve-no-subscription repo can be used for testing and non-production use. Its not recommended to run on production servers as these packages are not as comprehensively tested and validated prior to release. For my intended use, the free version will suffice.

Enter `nano /etc/apt/sources.list` to edit the repositories
```
deb http://ftp.debian.org/debian jessie main contrib

# PVE pve-no-subscription repository provided by proxmox.com, NOT recommended for production use
deb http://download.proxmox.com/debian jessie pve-no-subscription

# security updates
deb http://security.debian.org/ jessie/updates main contrib
```

`CTRL-X`, `Y` to Save and exit.

Enter `nano /etc/apt/sources.list.d/pve-enterprise.list` and comment the following line out by placing a hash (**#**) character at the beginning the line to disable and prevent errors in the logs the subscription only repo.s.
```
# deb https://enterprise.proxmox.com/debian jessie pve-enterprise
```

`CTRL-X`, `Y` to Save and exit.

It.s worth updating and upgrading your packages at this time with the following commands
```
apt-get update
apt-get upgrade
```

### Set NTP time servers

Its important to have correctly syncronised clocks across your networks to prevent any issues. Lets ensure our Proxmox server is contacting the correct server for NTP info. You can use any NTP server for your area/region for NTP synchronisation.

Enter `nano /etc/systemd/timesyncd.conf` and add the following line but specify your own NTP server here.
```
Servers=vn.pool.ntp.org
```

you can verify NTP is working by entering the command
```
timedatectl status
```

and you should see the following lines in the response verifying NTP is enabled and syncronized.
```
root@proxmox:~# timedatectl status
      Local time: Sat 2016-07-01 22:42:02 EDT
  Universal time: Sun 2016-07-02 02:42:02 UTC
        RTC time: Sun 2016-07-02 02:42:02
       Time zone: America/New_York (EDT, -0400)
     NTP enabled: yes
NTP synchronized: yes
 RTC in local TZ: no
      DST active: yes
```

### part3: Log in to the Proxmox GUI!

Finally. lets log into the web interface to complete our configuration. 
Open a browser and head to the GUI address as entered during your install and also as displayed on the console page.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-9.png "")

and enter your username and admin password
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-10.png "")

and assuming you don.t have a subscription, acknowledge the lack of subscription option.

#### Configure additional network interface

I now recommend editing the network configuration via the Proxmox web interface rather than directly in the `/etc/network/interfaces` file.

During installation, Proxmox configures the first interface found during install as its default interface, vmbr0. I use this interface for connecting to the Proxmox console, GUI and transferring backups of virtual machines to my NAS. I.ve added a second interface specifically for the virtual machines to use. This second interface is provided by a 10gig Intel x520 which is configured to supports two VLANs, VLAN 10 which is my management (MGMT) interface and VLAN 20 which is my general LAN. Depending on the intended use of the virtual machine I am creating, I.ll attach it to either my MGMT or LAN interface.

Navidate to ServerView > DataCenter > Proxmox and select Network.

#### Create vmbr10 / VLAN 10 - VL10_MGMT

Click Create and select Linux Bridge

![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-11.png "")
```
Name = vmbr10
IP address = empty
Subnet mask = empty
Gateway = empty
IPv6 address = empty
Prefix length = empty
Gateway = empty
Autostart = [x]
Vlan aware = [x]
Bridge ports = eth1.10
```

#### Create vmbr20 / VLAN 20 - VL20_VPN

Click Create and select Linux Bridge

![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-12.png "")
```
Name = vmbr20
IP address = empty
Subnet mask = empty
Gateway = empty
IPv6 address = empty
Prefix length = empty
Gateway = empty
Autostart = [x]
Vlan aware = [x]
Bridge ports = eth1.20
```

and here.s what the finished Network view should look like when you are finished.
![alt text](https://github.com/yoriyatana/VRR-on-KVM/blob/master/IMG/install-proxmox-13.png "")

And for completeness, here.s how this configuration translates into the `/etc/network/interfaces` configuration file.
```
$ cat /etc/network/interfaces
# network interface settings; autogenerated
# Please do NOT modify this file directly, unless you know what
# you're doing.

# If you want to manage part of the network configuration manually,
# please utilize the 'source' or 'source-directory' directives to do
# so.
# PVE will preserve these directives, but will NOT its network
# configuration from sourced files, so do not attempt to move any of
# the PVE managed interfaces into external files!

auto lo
iface lo inet loopback

iface eth0 inet manual

iface eth1 inet manual

iface eth2 inet manual

iface eth3 inet manual

auto vmbr0
iface vmbr0 inet static
  address  192.168.20.12
  netmask  255.255.255.0
  gateway  192.168.20.1
  bridge_ports eth0
  bridge_stp off
  bridge_fd 0

auto vmbr20
iface vmbr20 inet manual
  bridge_ports eth1.20
  bridge_stp off
  bridge_fd 0
  bridge_vlan_aware yes

auto vmbr10
iface vmbr10 inet manual
  bridge_ports eth1.10
  bridge_stp off
  bridge_fd 0
  bridge_vlan_aware yes
```
For a full description of this file and it contents, refer to the Debian network interface man pages

Save the file and reboot.

### Part4: Create vRR VM using command line interface
Copy the vRR image to the PVE server via SCP or SFTP

Rename it to the name you want to call the disk. *I.E. VMid 106 and disk called vm-106-disk-1.qcow2*
```
export VM=$(pvesh get /cluster/nextid | grep -oE '[0-9]*')
cp jinstall64-vrr-14.2R7.5-domestic.img vm-$VM-disk-1.qcow2
```

Create the new VM with the `qm` command. You must specify the VM name, memory, and image location. The amount of memory depends on your deployment.
```
qm create $VM --net0 e1000,bridge=vmbr0 --net1 e1000,bridge=vmbr1 --cores 4 --cpu host --name vRR --memory 8192 --bootdisk ide0 --serial0 socket
```

Create the directory for VM disk image
```
mkdir /var/lib/vz/images/$VM/
```

move your file in vm-$VM-disk-1.qcow2 to the correct location
```
cp vm-$VM-disk-1.qcow2 /var/lib/vz/images/$VM/
```

Update the disk image for vRR
```
qm set $VM --ide0  local:$VM/vm-$VM-disk-1.qcow2
```

Start your machine
```
qm start $VM
```

Connect to the VM console
```
qm terminal $VM
```

When the installation is completed, the login prompt appears:
```
Amnesiac (ttyd0)

login:
```

Log in from the console with the **username:** `root` and `no password`. Type cli to access the Junos OS CLI.

Verify that your VM is installed using the show interfaces terse command. The added interfaces appear as em interfaces.

To disconnect from the console, press `Ctrl + o`.
```
press Ctrl + o
```

## Source

* https://nguvu.org/proxmox/proxmox-install/

