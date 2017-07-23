# Juniper vRR on KVM

Tutorial for installing the Juniper vRR on KVM

## Getting Started

These instructions will get you a copy of the project up and running on your local machine for development and testing purposes. See deployment for notes on how to deploy the project on a live system.

### Prerequisites

What things you need to install the software and how to install them

```
VM with CentOS 7.2
	 8GB RAM, 4 vCPUs and two vNICs
	 Also the VM is enabled to support hypervisor applications within the VM.
jinstall64-vrr-14.2R7.5-domestic.img images
```

### Installing

#### Part1: KVM installation and preparation

At this point CentOS Server has been freshly installed, and the option to install virtualisation was selected during setup.

Make sure that your system has the hardware virtualization extensions: For Intel-based hosts, verify the CPU virtualization extension [vmx] are available using following command.
```
grep -e 'vmx' /proc/cpuinfo
```

For AMD-based hosts, verify the CPU virtualization extension [svm] are available.
```
grep -e 'svm' /proc/cpuinfo
```

If there is no output make sure that virtualization extensions is enabled in BIOS. Verify that KVM modules are loaded in the kernel “it should be loaded by default”.
```
lsmod | grep kvm
```

Before starting , you will need the root account or non-root user with sudo privileges configured on your system and also make sure that your system is up-to-date.
```
yum update
```

Make sure that Selinux be in Permissive mode.
```
setenforce 0
```

We will install qemu-kvm and qemu-img packages at first. These packages provide the user-level KVM and disk image manager.
```
yum install -y qemu-kvm qemu-img
```

Now, you have the minimum requirement to deploy virtual platform on your host, but we also still have useful tools to administrate our platform such as:
```
virt-manager provides a GUI tool to administrate your virtual machines.
libvirt-client provides a CL tool to administrate your virtual environment this tool called virsh.
virt-install provides the command “virt-install” to create your virtual machines from CLI.
libvirt provides the server and host side libraries for interacting with hypervisors and host systems.
Let’s install these above tools using the following command.
```

```
yum install -y virt-manager libvirt libvirt-python libvirt-client 
```

For RHEL/CentOS7 users, also still having additional package groups such as: Virtualization Client, Virtualization Platform and Virtualization Tools to install.
```
yum groupinstall virtualization-client virtualization-platform virtualization-tools
```

The virtualization daemon which manage all of the platform is “libvirtd”. lets restart it.
```
systemctl restart libvirtd
```

After restarting the daemon, then check its status by running following command.
```
systemctl status libvirtd 
```

Now, lets switch to the next section to create our virtual machines.

#### Part2: Create vRR VM using KVM
Copy the vRR image to the libvirt directory and rename it with the name of your VM
```
cp  jinstall64-vrr-14.2R7.5-domestic.img /var/lib/libvirt/images/vRR.img
```

Install the vRR VM using the virt-install command. You must specify the VM name, memory, and image location. The amount of memory depends on your deployment.
```
virt-install 	--name vRR-VM1 \
				--ram 9192 \
				--disk path=/var/lib/libvirt/images/vRR.img \
				--cpu host --vcpus 4 \
				--os-variant none \
				--graphics none \
				--import \
				--network bridge=bridge0,model=e1000 \
				--network bridge=bridge1,model=e1000
```

Connect to the VM console using the virsh console vm-name command.
```
virsh console vRR-VM1
```

When the installation is completed, the login prompt appears:
```
Amnesiac (ttyd0)

login:
```

To disconnect from the console, press Ctrl + ].
```
press Ctrl + ]
```


Log in from the console with the username root and no password. Type cli to access the Junos OS CLI.

Verify that your VM is installed using the show interfaces terse command. The added interfaces appear as em interfaces.

## Source

* Juniper official website

