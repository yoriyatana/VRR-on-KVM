I just found a solution, forgot to update my forum post. I was trying to use the default "local-lvm" storage, which only supports `*.raw`. My solution was to take the default `/etc/pve/storage.cfg`:

```
dir: local
        path /var/lib/vz
        content iso,vztmpl,backup

lvmthin: local-lvm
        thinpool data
        vgname pve
        content rootdir,images
And add images to "local" storage:
```

```
dir: local
        path /var/lib/vz
        content iso,vztmpl,backup,images

lvmthin: local-lvm
        thinpool data
        vgname pve
        content rootdir,images
```
There is already an images folder within `/var/lib/vz` (the default local storage directory), so why is storing images there disabled by default?

## Source
https://forum.proxmox.com/threads/how-to-migrate-a-virtual-machine-image-into-local-lvm.28751/
