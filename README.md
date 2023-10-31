# Proxmox Install Documented


## LSI 9211-8i flashed to IT mode
Use DOS to flash firmwware because efi mode does not support things properly  
https://www.broadcom.com/support/knowledgebase/1211161501344/flashing-firmware-and-bios-on-lsi-sas-hbas  
https://nguvu.org/freenas/Convert-LSI-HBA-card-to-IT-mode/  


## ASUS PRIME x299-A II
Turn on Advanced &rarr; CPU Storage Configuration &rarr; Choose HBA PCIE Slot &rarr; Hyper M.2 X16 Data (Non-VROC)  
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
zfs set autotrim=on {{pool name}}
```

### ZFS Pool & NFS Config
```
zfs create ZFS/nextcloud
zfs create ZFS/kubernetes
zfs create ZFS/proxmox

zfs set quota=750G ZFS/nextcloud
zfs set quota=250G ZFS/kubernetes
zfs set quota=100G ZFS/iso
zfs set quota=100G ZFS/proxmox-vm
zfs set quota=100G ZFS/proxmox-other

apt install nfs-kernel-server //ensure that nfs-common is off

# add no_root_squash if need to manually move files into nextcloud directories (need it to allow chown for www-data user) https://serverfault.com/questions/212178/chown-on-a-mounted-nfs-partition-gives-operation-not-permitted
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/nextcloud
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/kubernetes
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/iso
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/proxmox-other

# Add nextcloud & kubernetes as ZFS storage
# Datacenter -> Storage -> ZFS -> Select Dataset, Thin-Provisioning

# Add Proxmox as NFS server
# Datacenter -> Storage -> NFS
```

## Proxmox Specific
### Accidentally Wiping LVM before properly destroying it
Leaves ghost LVM that can't be removed via GUI  
Gather pve name from ```pvesm status```  
Delete with ```pvesm remove <<pve name>>```  
https://192.168.1.208:8006/pve-docs/chapter-pvesm.html#chapter_storage


## Application Specific

### Nextcloud

#### Tips
![image](https://github.com/knta7/proxmox/assets/107367656/09373160-9136-47fc-9039-9ac46ccf551c)

#### Manual Backup (Offical Documentation)
https://docs.nextcloud.com/server/latest/admin_manual/maintenance/migrating.html  
DID NOT WORK AT ALL- most likely because nextcloud versions were too different (database issues)  
~~Copy data from /mnt/ncdata/{{username}}  
Copy nextcloud dir from /var/www/nextcloud
Postgres backup  
Get postgres user and pass from /var/www/nextcloud/config/config.php  
Get db backup ```PGPASSWORD="{{password here}}" pg_dump nextcloud_db -h 127.0.0.1 -U {{username here}} -f nextcloud-sqlbkp_`date +"%Y%m%d"`.bak```~~

#### Working Backup Solution
1. Copy data from ```/mnt/ncdata/{{username}}```
2. Create brand new nextcloud instance
3. Create users manually so folders are in ```/mnt/ncdata/{{username}}```
    1. OPTIONAL: Log in as each user to produce first time login files and then delete manually
5. Install nfs ```apt install nfs-common```
6. NOTE: Ensure no_root_squash is on nfs server settings- because you need to:
    1. ```chown -R www-data:www-data /mnt/{{nfs mount folder}}```
    2. ```chmod -R 0770 /mnt/{{nfs mount folder}}```
    3. If nfs is already mounted, either remount or just restart (after adding fstab entry)
    4. Mount nfs volume with copied data ```mount -t nfs {{nfs-server-ip}}:{{nfs-server-export-folder}} {{nfs mount folder}}```
7. Add ```/etc/fstab``` entry ```{{nfs-server-ip}}:{{nfs-server-export-folder}} {{desired mount folder}} defaults 0 0```
8. Copy over hidden files ```.ocdata, .htaccess```
9. Change following settings in ```/var/www/nextcloud/config/config.php```
    1. ```maintanence => false``` &rarr; ```maintainence => true```
    2. ```'datadirectory' => '/mnt/ncdata'``` &rarr; ```'datadirectory' => '/mnt/{{nfs server}}'```
10. Run ```sudo -u www-data /var/www/nextcloud/occ files:scan --all```
    1. If nextcloud errors to change file permissions to 0770, add ```'check_data_directory_permissions' => false``` to ```/var/www/nextcloud/config/config.php```
