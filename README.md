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
    - [ ] Create new HDD ZFS
        - [ ] Migrate large uncompressable + sequential media to HDD ZFS
        - [ ] Update K8s deployments and VMs from old nfs to new nfs
- [x] Setup K8s cluster
    - [ ] Sandbox
    - [x] Prod
        - [ ] Setup Prometheus
    - [ ] Monitoring
        - [ ] Disk (IO, Temps, SMART)
        - [ ] Internet (Speed, Latency)
        - [ ] Component Temps
- [ ] Migrate data in old esxi ZFS to new proxmox ZFS
    - [x] Vaultwarden
        - [x] Config
        - [x] Postgres Data
    - [x] Jellyfin
        - [x] Config
        - [x] Media
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

#ensure that nfs-common is off- nfs-common does not create nfs server properly
apt install nfs-kernel-server 
```

See physical sector size using `fdisk -l`  
Set ashift equal to physical sector size (2^(ashiftvalue) &rarr; 2^12 = 4096 == sector size)  

### ZFS Pool & NFS Config
#### SSD
```
zfs create ZFS/nextcloud
zfs create ZFS/kubernetes
zfs create ZFS/jellyfin
zfs create ZFS/jellyfin/config
zfs create ZFS/jellyfin/media
zfs create ZFS/vaultwarden
zfs create ZFS/vaultwarden/config
zfs create ZFS/vaultwarden/postgres-data
zfs create ZFS/gitea
zfs create ZFS/gitea/config
zfs create ZFS/gitea/data
zfs create ZFS/gitea/postgres-data
zfs create ZFS/iso
zfs create ZFS/proxmox-vm
zfs create ZFS/proxmox-other
zfs create ZFS/game-data
zfs create ZFS/game-data/terraria
zfs create ZFS/test
zfs create ZFS/test/vaultwarden/config
zfs create ZFS/test/vaultwarden/postgres-data
zfs create ZFS/cfg

zfs set quota=1000G ZFS/nextcloud
zfs set quota=250G ZFS/kubernetes
zfs set quota=5G ZFS/jellyfin
zfs set quota=5G ZFS/jellyfin/config
zfs set quota=1G ZFS/vaultwarden
zfs set quota=0.5G ZFS/vaultwarden/config
zfs set quota=0.5G ZFS/vaultwarden/postgres-data
zfs set quota=5G ZFS/gitea
zfs set quota=0.5G ZFS/gitea/config
zfs set quota=0.5G ZFS/gitea/data
zfs set quota=4G ZFS/gitea/postgres-data
zfs set quota=100G ZFS/iso
zfs set quota=100G ZFS/proxmox-vm
zfs set quota=100G ZFS/proxmox-other
zfs set quota=150G ZFS/game-data
zfs set quota=50G ZFS/game-data/terraria
zfs set quota=50G ZFS/test
zfs set quota=1G ZFS/cfg

# add no_root_squash if need to allow chown for files (eg nextcloud and www-data user) https://serverfault.com/questions/212178/chown-on-a-mounted-nfs-partition-gives-operation-not-permitted
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/nextcloud
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/kubernetes
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/jellyfin
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/vaultwarden
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/jellyfin/config
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/vaultwarden/config
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/vaultwarden/postgres-data # postgres user chown's the folder
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/gitea/config
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/gitea/data
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/gitea/postgres-data
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/iso
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/proxmox-other
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/game-data/terraria
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/test
zfs set sharenfs='rw=@192.168.1.1/16,sync' ZFS/test/vaultwarden/config
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/test/vaultwarden/postgres-data
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' ZFS/cfg

# Add nextcloud & kubernetes as ZFS storage
# Datacenter -> Storage -> ZFS -> Select Dataset, Thin-Provisioning

# Add Proxmox as NFS server
# Datacenter -> Storage -> NFS
```

#### HDD
```
zfs create HDD/jellyfin
zfs create HDD/jellyfin/media
zfs create HDD/iso
zfs create HDD/kubernetes
zfs create HDD/file-server
zfs create HDD/backups
zfs create HDD/backups/postgres

zfs set quota=10T HDD/jellyfin
zfs set quota=10T HDD/jellyfin/media
zfs set quota=500G HDD/iso
zfs set quota=1T HDD/kubernetes
zfs set quota=100G HDD/file-server
zfs set quota=100G HDD/backups
zfs set quota=50G HDD/backups/postgres

zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' HDD/jellyfin/media
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' HDD/iso
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' HDD/kubernetes
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' HDD/file-server
zfs set sharenfs='rw=@192.168.1.1/16,sync,no_root_squash' HDD/backups/postgres
```

## Proxmox Specific
### Accidentally Wiping LVM before properly destroying it
Leaves ghost LVM that can't be removed via GUI  
Gather pve name from ```pvesm status```  
Delete with ```pvesm remove <<pve name>>```  
https://192.168.1.208:8006/pve-docs/chapter-pvesm.html#chapter_storage

### Remove directory mounts
Must do it via console
```
lsblk
umount <mount point>
rm /etc/systemd/system/mnt-pve-testdir.mount

#Gather pve name from
pvesm status

#Delete with 
pvesm remove <pve name>
```

### Create Kubernetes Template Image
```
# Taken from https://github.com/UntouchedWagons/Ubuntu-CloudInit-Docs
# Download premade cloud-init image

#!/bin/bash
VM_ID=$1
storage=$2

if [[ -z $1 || -z $2 ]]; then
        echo "Variable not set"
        exit 1
fi

# Create vm
qm create $VM_ID --name "ubuntu-2204-cloudinit-template" --ostype l26 \
    --memory 2048 \
    --agent 1 \
    --bios ovmf --machine q35 --efidisk0 $storage:0,pre-enrolled-keys=0 \
    --cpu host --socket 1 --cores 2 \
    --net0 virtio,bridge=vmbr0

qm importdisk $VM_ID jammy-server-cloudimg-amd64.img $storage
qm set $VM_ID --scsihw virtio-scsi-pci --virtio0 $storage:$VM_ID/vm-$VM_ID-disk-1.raw
qm set $VM_ID --boot c --bootdisk virtio0
qm set $VM_ID --ide2 $storage:cloudinit

# Create vendor.yaml to run once after boot
cat << EOF | tee /var/lib/vz/snippets/vendor.yaml
#cloud-config
runcmd:
    - apt update
    - apt install -y qemu-guest-agent nfs-common
    - systemctl start qemu-guest-agent
    - swapoff -a
    - ufw disable
    - echo \`date\` > /var/log/cloud-init-completed.log
    - reboot
# Taken from https://forum.proxmox.com/threads/combining-custom-cloud-init-with-auto-generated.59008/page-3#post-428772
EOF

# Configuring CloudInit
qm set $VM_ID --cicustom "vendor=local:snippets/vendor.yaml"
qm set $VM_ID --tags ubuntu-template,22.04,cloudinit
qm set $VM_ID --ciuser px-user
qm set $VM_ID --cipassword $(openssl passwd -6 $CLEARTEXT_PASSWORD)
qm set $VM_ID --sshkeys ~/.ssh/authorized_keys
qm set $VM_ID --ipconfig0 ip=dhcp

# Convert to template
#qm template $VM_ID
```

***NOTE***  
**If there are common packages tha need to be installed and you want to avoid the template re-dl every time (aka set up golden image), run VM before converting to template and manually install- remember to truncate /etc/machine-id**
#### Docker CRI
```
# Packages manually installed onto template to save time:
## Docker
sudo apt-get update -y
sudo apt-get install -y ca-certificates curl gnupg
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

echo "deb [arch="$(dpkg --print-architecture)" signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  "$(. /etc/os-release && echo "$VERSION_CODENAME")" stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt-get update -y

sudo apt-get install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y

## kubeadm
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

## cri-dockerd
wget https://github.com/Mirantis/cri-dockerd/releases/download/v0.3.4/cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb
dpkg -i cri-dockerd_0.3.4.3-0.ubuntu-jammy_amd64.deb
systemctl daemon-reload
systemctl enable --now cri-docker.socket

## On kubernetes master node- create join token
kubeadm token create --ttl 0

## On template cloud init- include following line in vendor.yaml
cat << EOF | tee /var/lib/vz/snippets/vendor.yaml
#cloud-config
runcmd:
    - kubeadm join 192.168.1.252:6443 --token TOKEN_FROM_BEFORE_HERE --discovery-token-ca-cert-hash CA_CERT_HERE --cri-socket=unix:///var/run/cri-dockerd.sock
# Taken from https://forum.proxmox.com/threads/combining-custom-cloud-init-with-auto-generated.59008/page-3#post-428772
EOF

## Apply and clone VM and test before finally converting to template

# Run the below command which truncates /etc/machine-id and ensures every machine gets new IP every time from cloning template
echo -n > /etc/machine-id
```

#### ContainerD CRI
```
sudo modprobe overlay
sudo modprobe br_netfilter
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo apt-get update
sudo apt-get install -y containerd

sudo mkdir -p /etc/containerd
sudo containerd config default | sudo tee /etc/containerd/config.toml

# Edit /etc/containerd/config.toml
# find the [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options] section and change systemdcgroup to true
# ex: SystemdCgroup = true

sudo systemctl restart containerd
sudo systemctl enable containerd

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF
sudo sysctl --system

sudo swapoff -a
sudo ufw disable

# Use iptables-legacy (did not try)
sudo update-alternatives --config iptables

## kubeadm
sudo apt-get install -y apt-transport-https ca-certificates curl gpg
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.28/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
echo 'deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.28/deb/ /' | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```
#### Post Manual Package Install
```
sudo cloud-init clean --logs

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

### Deluge + Wireguard (VM)
https://mullvad.net/en/help/wireguard-and-mullvad-vpn/
1. `curl -o mullvad-wg.sh https://raw.githubusercontent.com/mullvad/mullvad-wg.sh/main/mullvad-wg.sh`
2. `chmod +x ./mullvad-wg.sh && ./mullvad-wg.sh`
3. `apt install openresolv`
4. `cd /etc/wireguard`
5. `wg-quick up us-nyc-wg-501.conf`
6. `curl https://am.i.mullvad.net/connected`
7. `wg-quick down us-nyc-wg-501.conf`
8. `systemctl enable wg-quick@us-nyc-wg-501`
9. Add to `/etc/fstab` `192.168.1.208:/ZFS/jellyfin/media /media defaults 0 0`

### Arr Setup
https://wiki.servarr.com/install-script


### Kubernetes


### Monitoring
#### Node Exporter Including Disk Metrics
```
wget https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
tar xvfs node_exporter-1.6.1.linux-amd64.tar.gz
cd node_exporter-1.6.1.linux-amd64/
cp node_exporter /usr/bin/
```
Create systemd service
```
cat /etc/systemd/system/node-exporter.service
[Unit]
Description=Promtheus Node Exporter
StartLimitIntervalSec=0

[Service]
Type=simple
Restart=always
RestartSec=1
User=root
ExecStart=/usr/bin/node_exporter --collector.textfile.directory=/var/lib/node_exporter/textfile_collector

[Install]
WantedBy=multi-user.target
```
Using https://github.com/micha37-martins/S.M.A.R.T-disk-monitoring-for-Prometheus  
```
# Download smartmon.sh from github
# setup crontab

crontab -e

*/5 * * * * /usr/local/bin/smartmon.sh > /var/lib/node_exporter/textfile_collector/smart_metrics.prom

service cron reload
```

#### KProximate
- If you already injected SSH key into VM, disable SSH key injection otherwise you get 400 Parameter Validation error
