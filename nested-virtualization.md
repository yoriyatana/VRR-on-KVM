#Enable Nested Hardware-assisted Virtualization
To be done on the physical PVE host (or any other hypervisor).

To have nested hardware-assisted virtualization, you have to:
```
use AMD cpu or very recent Intel one
use kernel >= 3.10 (is always the case in Proxmox VE 4.x)
enable nested support
```

to check if is enabled do ("kvm_intel" for intel cpu, "kvm_amd" for AMD)
```
 root@proxmox:~# cat /sys/module/kvm_intel/parameters/nested   
N
```

N means it's not, to enable ("kvm-intel" for intel):
```
 # echo "options kvm-intel nested=Y" > /etc/modprobe.d/kvm-intel.conf
```

(or "kvm-amd" for AMD, note the 1 instead of Y):
```
 # echo "options kvm-amd nested=1" > /etc/modprobe.d/kvm-amd.conf
```
 
and reboot or reload the kernel module
```
modprobe -r kvm_intel
modprobe kvm_intel
```
check again
```
 root@proxmox:~# cat /sys/module/kvm_intel/parameters/nested                    
 Y
```

(pay attention where the dash "-" is used, and where it's underscore "_" instead)

Then create a guest where you install e.g. Proxmox as nested Virtualization Environment.

set the CPU type to "host"
in case of AMD CPU: add also the following in the configuration file:
```
 args: -cpu host,+svm
```
Once installed the guest OS, if GNU/Linux you can enter and verify that the hardware virtualization support is enabled by doing
```
 root@guest1# egrep '(vmx|svm)' --color=always /proc/cpuinfo
```