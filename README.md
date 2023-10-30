# Proxmox Install Documented


## LSI 9211-8i flashed to IT mode
Use DOS to flash firmwware because efi mode does not support things properly  
https://www.broadcom.com/support/knowledgebase/1211161501344/flashing-firmware-and-bios-on-lsi-sas-hbas  
https://nguvu.org/freenas/Convert-LSI-HBA-card-to-IT-mode/  


## ASUS PRIME x299-A II
Turn on Advanced/CPU Storage Configuration/Hyper M.2 X16 Data (Non-VROC)  
Turn on CSM  
Turn off FastBoot  

## ZFS
### Troubleshooting
```
zpool get all
```

### ZFS General Settings
```
# For SSD only storage
zfs set autotrim=on <<pool name>>
```

### ZFS Pool & NFS Config
```
zfs create ZFS/nextcloud
zfs create ZFS/kubernetes
zfs create ZFS/proxmox

zfs set quota=750G ZFS/nextcloud
zfs set quota=250G ZFS/kubernetes
zfs set quota=100G ZFS/proxmox

apt install nfs-kernel-server //ensure that nfs-common is off

zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/nextcloud
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/kubernetes
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/proxmox
```

## Proxmox Specific
### Accidentally Wiping LVM before properly destroying it
Leaves ghost LVM that can't be removed via GUI  
Gather pve name from ```pvesm status```  
Delete with ```pvesm remove <<pve name>>```  
https://192.168.1.208:8006/pve-docs/chapter-pvesm.html#chapter_storage
