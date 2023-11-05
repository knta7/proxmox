# Proxmox Install Documented


## To-Do
- [x] Create ZFS datasets & NFS shares
    - [x] Kubernetes
    - [x] Jellyfin
        - [x] Config
        - [x] Media
    - [x] Vaultwarden
        - [x] Config
        - [x] Postgres Data
- [x] Setup K8s cluster
    - [ ] Sandbox
    - [x] Prod
- [ ] Migrate data in old esxi ZFS to new proxmox ZFS
    - [x] Vaultwarden
        - [x] Config
        - [x] Postgres Data
    - [ ] Jellyfin
        - [x] Config
        - [ ] Media
    - [ ] Jenkins

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
zfs create ZFS/jellyfin
zfs create ZFS/jellyfin/config
zfs create ZFS/jellyfin/media
zfs create ZFS/vaultwarden
zfs create ZFS/vaultwarden/config
zfs create ZFS/vaultwarden/postgres-data
zfs create ZFS/iso
zfs create ZFS/proxmox-vm
zfs create ZFS/proxmox-other

zfs set quota=750G ZFS/nextcloud
zfs set quota=250G ZFS/kubernetes
zfs set quota=500G ZFS/jellyfin
zfs set quota=5G ZFS/jellyfin/config
zfs set quota=495G ZFS/jellyfin/media
zfs set quota=1G ZFS/vaultwarden
zfs set quota=0.5G ZFS/vaultwarden/config
zfs set quota=0.5G ZFS/vaultwarden/postgres-data
zfs set quota=100G ZFS/iso
zfs set quota=100G ZFS/proxmox-vm
zfs set quota=100G ZFS/proxmox-other

apt install nfs-kernel-server #ensure that nfs-common is off

# add no_root_squash if need to manually move files into nextcloud directories (need it to allow chown for www-data user) https://serverfault.com/questions/212178/chown-on-a-mounted-nfs-partition-gives-operation-not-permitted
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/nextcloud
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/kubernetes
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/jellyfin
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/vaultwarden
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/jellyfin/config
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/jellyfin/media
zfs set sharenfs='rw=@192.168.1.0/24,sync' ZFS/vaultwarden/config
zfs set sharenfs='rw=@192.168.1.0/24,sync,no_root_squash' ZFS/vaultwarden/postgres-data # postgres user chown's the folder
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

### Create Kubernetes Template Image
```
# Taken from https://github.com/UntouchedWagons/Ubuntu-CloudInit-Docs
# Download premade cloud-init image
wget -q https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Resize disk
qemu-img resize jammy-server-cloudimg-amd64.img 32G

# Create vm
qm create 9001 --name "ubuntu-2204-cloudinit-template" --ostype l26 \
    --memory 2048 \
    --agent 1 \
    --bios ovmf --machine q35 --efidisk0 proxmox-vm:0,pre-enrolled-keys=0 \
    --cpu host --socket 1 --cores 1 \
    --vga serial0 --serial0 socket  \
    --net0 virtio,bridge=vmbr0

qm importdisk 9001 jammy-server-cloudimg-amd64.img proxmox-vm
qm set 9001 --scsihw virtio-scsi-pci --virtio0 proxmox-vm:vm-9001-disk-1,discard=on
qm set 9001 --boot c --bootdisk virtio0
qm set 9001 --ide2 proxmox-vm:cloudinit

# Create vendor.yaml to run once after boot
cat << EOF | tee /var/lib/vz/snippets/vendor.yaml
#cloud-config
runcmd:
    - apt update
    - apt install -y qemu-guest-agent
    - systemctl start qemu-guest-agent
    - echo 'test' > /var/log/testere.log
    - reboot
# Taken from https://forum.proxmox.com/threads/combining-custom-cloud-init-with-auto-generated.59008/page-3#post-428772
EOF

# Configuring CloudInit
qm set 9001 --cicustom "vendor=local:snippets/vendor.yaml"
qm set 9001 --tags ubuntu-template,22.04,cloudinit
qm set 9001 --ciuser proxmox-user
qm set 9001 --cipassword $(openssl passwd -6 $CLEARTEXT_PASSWORD)
qm set 9001 --sshkeys ~/.ssh/authorized_keys
qm set 9001 --ipconfig0 ip=dhcp

# Convert to template
qm template 9001

# If there are common packages tha need to be installed and you want to avoid re-dl every time
# Run the below command to ensure machine gets new IP every time from template
echo -n > /etc/machine-id
```

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

