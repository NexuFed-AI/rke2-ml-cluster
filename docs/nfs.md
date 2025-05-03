# Setup NFS
This tutorial describes how to setup a fast NVME SSD NFS server with RDMA support on Debian 11. The server is used to store the data for the Kubernetes cluster. The server is connected to the cluster via RDMA and NFSv4.2.

When considering other storage solutions, such as NVIDIA AIStore, this perfomance guide might be worth a look: [NVIDIA AI Storage Performance](https://aiatscale.org/docs/performance).

## Install Debian
- Create a bootable USB stick with [Debian Live](https://cdimage.debian.org/debian-cd/current-live/amd64/iso-hybrid/) or [Debian](https://cdimage.debian.org/debian-cd/current/amd64/iso-cd/) using [Rufus](https://rufus.ie/).
- Boot from the USB stick and install Debian.
- Select:
    - Language: English
    - Location: Other -> Europe -> Germany
    - Locale: United States - en_US.UTF-8
    - Keyboard: German
    - Hostname: <label_on_pc_case>
    - Domain name: ml-cluster.local
    - Root password: password
    - Full name for the new user: admin
    - Username for your account: admin
    - Choose a password for the new user: password
    - Force UEFI installation: Yes
    - Partitioning method: Guided - use entire disk
    - Partitioning scheme: All files in one partition (recommended for new users)
    - Install
    - Use a network mirror: Yes
    - Reboot

### Configure BIOS
- Turn on Wake-on-LAN (PCIe)
- Turn on boot on power (Restore AC Power Loss)
- (Turn off secure boot)
- (Turn off IOMMU)

### Configure Debian
- Login as `admin`
- Open a terminal
- Install dependencies follow the following steps:
    - `su -`
        - Run `apt update`
        - Install `apt install git tmux net-tools htop sshfs openssh-server ufw`
    - Get IP address if you want to login via SSH from your main computer
        - `ip a` or `ifconfig`
    - Add `admin` to the `sudo` group
        - `su -`
        - `/usr/sbin/usermod -aG sudo admin`
        - exit
        - Logout and login again
        - `sudo whoami`
    - Remove password for `sudo`
        - sudo nano `/etc/sudoers`
        - Add `admin ALL=(ALL) NOPASSWD:ALL` to the end of the file
        - Save and exit
    - Add `contrib` and `non-free` to `/etc/apt/sources.list`
        - `sudo nano /etc/apt/sources.list`
        - Add `contrib non-free non-free-firmware` to the end of every line that starts with `deb`
        - Save and exit (Ctrl+X, Y, Enter)
    - Disable sleep mode
        - `sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target`
        - Check status `sudo systemctl status sleep.target suspend.target hibernate.target hybrid-sleep.target`

## Create RAID 0
- Install dependencies
```bash
sudo apt update && sudo apt upgrade
sudo apt install parted mdadm lvm2 nvme-cli hdparm
export PATH=$PATH:/usr/sbin
```

- Optimize NVME
```bash	
# Get the available disks
lsblk -o NAME,SIZE,FSTYPE,TYPE,MOUNTPOINT

# Check all NVMEs
sudo /usr/sbin/nvme list
sudo /usr/sbin/nvme show-topology
sudo lspci -vv -s 0000:01:00.0

# Find best block-size
sudo /usr/sbin/nvme id-ns -H /dev/nvme0n1 | grep "Relative Performance"

# Set the LBA Format to index 1 (4K) Better to be in use
sudo /usr/sbin/nvme format --lbaf=1 /dev/nvme0n1
sudo /usr/sbin/nvme format --lbaf=1 /dev/nvme1n1
sudo /usr/sbin/nvme format --lbaf=1 /dev/nvme2n1
sudo /usr/sbin/nvme format --lbaf=1 /dev/nvme3n1
sudo /usr/sbin/nvme format --lbaf=1 /dev/nvme4n1
```
- Create a LVM software RAID 0 with 5 NVME SSDs
```bash
# Create pyhsical volumes
sudo pvcreate /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1
sudo pvdisplay
sudo pvs
sudo pvscan

# Create volume group
sudo vgcreate nvme-raid /dev/nvme0n1 /dev/nvme1n1 /dev/nvme2n1 /dev/nvme3n1 /dev/nvme4n1
sudo vgs
sudo vgdisplay nvme-raid
sudo vgscan
# vgextend nvme-raid /dev/nvmeXn1
# vgrename nvme-raid new-name

# Create logical volume
sudo lvcreate --type raid0 --stripes 5 -l 100%FREE -n raid nvme-raid
sudo lvs
# sudo lvremove /dev/nvme-raid/raid
# sudo lvconvert --type raid0 --stripes 5 nvme-raid/raid
# sudo lvs -a -o name,copy_percent,devices nvme-raid

# Check the layout
sudo lvs -a -o +devices,segtype nvme-raid

# Max RAID performance
sudo hdparm -Tt /dev/nvme-raid/raid
```
- Create a filesystem and mount the RAID 0
```bash	
# Get best block size
sudo tune2fs -l /dev/nvme-raid/raid | grep "RAID stride"
sudo tune2fs -l /dev/nvme-raid/raid | grep "RAID stripe width"

# Create Filesystem
# You might also consider XFS instead of ext4
sudo mkfs.ext4 -F /dev/nvme-raid/raid
# sudo mkfs.ext4 -b 4096 -E stride=128,stripe-width=640 -F /dev/md0

# Create Mound Point
sudo mkdir -p /mnt/raid
sudo mount /dev/nvme-raid/raid /mnt/raid
sudo umount /mnt/raid

# Show the filesystems
df -h -x devtmpfs -x tmpfs

# Automatic mounting on boot
echo '/dev/nvme-raid/raid /mnt/raid ext4 defaults,noatime,nodiratime,nobarrier,discard,data=writeback,nofail 0 0' | sudo tee -a /etc/fstab
# default = rw, suid, dev, exec, auto, nouser, async

sudo systemctl daemon-reload
sudo mount -a
```

## Install NFS and configure

### Install NFS
```bash
# On Server
sudo apt update
sudo apt install nfs-kernel-server

# Create folder to share
sudo mkdir -p /mnt/raid/k8s
sudo mkdir -p /mnt/raid/home

# Create folder for k8s
sudo mkdir -p /mnt/raid/k8s/{k8s_dev,k8s_hpc,k8s_web}

# Bind the folder to the NFS
sudo mkdir -p /exports/k8s
sudo mkdir -p /exports/home

sudo nano /etc/fstab
/mnt/raid/k8s        /exports/k8s        none    bind    0 0
/mnt/raid/home       /exports/home       none    bind    0 0

sudo systemctl daemon-reload
sudo sudo mount -a

# Set rights for raid mont point
sudo chown -R nobody:nogroup /mnt/raid
sudo chmod -R 777 /mnt/raid
sudo chown -R nobody:nogroup /exports
sudo chmod -R 777 /exports

# Add more CPU Cores to 32
sudo sed -i '/threads/{s/^# threads=8/threads=8/; s/threads=8/threads=32/}' /etc/nfs.conf
cat /etc/nfs.conf | grep threads

# Set folder to share
# https://linux.die.net/man/5/exports
# Try avoiding no_root_squash
sudo nano /etc/exports
/exports/k8s 10.0.0.0/25(rw,async,insecure,no_subtree_check,no_root_squash,fsid=1)
/exports/home 10.0.0.0/25(rw,async,insecure,no_subtree_check,no_root_squash,anonuid=65534,anongid=65534,fsid=2)

# Restart NFS and apply changes
sudo exportfs -ra
sudo systemctl enable nfs-kernel-server
sudo systemctl restart nfs-kernel-server
sudo exportfs -v
```

### Install RDMA
```bash
# The following needs to be done for both, server and client
sudo apt update
sudo apt install \
    iproute2 rdma-core libibverbs1 librdmacm1 \
    libibmad5 libibumad3 librdmacm1 ibverbs-providers rdmacm-utils \
    infiniband-diags libfabric1 ibverbs-utils

# Verify that Soft-RoCE is installed
sudo modinfo rdma_rxe
# Load Kernel Module
sudo modprobe rdma_rxe

# Add the modules to be loaded on reboot
sudo nano /etc/modules-load.d/rdma.conf
rdma_cm
rdma_ucm
ib_uverbs
ib_iser
rdma_cm
ib_umad
ib_ipoib
ib_cm
ib_core
rdma_rxe
rpcrdma

# Veryfiy that the Kernel modules are loaded
lsmod | grep '\(^ib\|^rdma\)'

# Reboot
sudo reboot

# Add the Links to the ethernet devices
# ip link show
for iface in $(ls /sys/class/net | grep -E '^e'); do echo -n "$iface: "; ip -o -4 addr show $iface | awk '{print $4}' | xargs echo -n "IP="; cat /sys/class/net/$iface/speed 2>/dev/null | xargs echo " Speed=" || echo " Speed=Not Available"; done

# In this case, we have eno1 and enp3s0
sudo rdma link add rxe_eth2G type rxe netdev eno1
sudo rdma link add rxe_eth10G type rxe netdev enp3s0

# See if devices are present
sudo rdma link show
ibv_devices
```
- Test the connection
```bash
# Test the connection
sudo apt install infiniband-diags ibutils ibverbs-utils perftest qperf fio

# On server
sudo rping -s -v -C 3
ibv_rc_pingpong
qperf
ib_send_bw
ib_send_lat

# On client
sudo rping -c -a 10.0.0.10 -v -C 3
ibv_rc_pingpong 10.0.0.10
qperf -cm1 10.0.0.10 rc_bw
ib_send_bw 10.0.0.10
ib_send_lat 10.0.0.10
```

### Enable RDMA on NFS
```bash
# On Server 
sudo sed -i '/rdma/{s/^#//; s/rdma=n/rdma=y/}' /etc/nfs.conf
grep -v ^# /etc/nfs.conf

sudo exportfs -ra
sudo systemctl restart nfs-kernel-server

# Check, if rdma port is present (20049)
cat /proc/fs/nfsd/portlist
```

## Mount NFS on Client

### Test mount options
```bash
sudo mkdir -p /mnt/nfs
sudo chown -R nobody:nogroup /mnt/nfs
sudo chmod -R 777 /mnt/nfs

sudo mount -t nfs -o defaults,nfsvers=4,rw,intr,nofail,nodiratime,noatime,async,x-systemd.automount,_netdev,fsid=0,no_root_squash,no_subtree_check 10.0.0.10:/ /mnt/nfs

sudo mount -t nfs -o fsid=0,rw,async,insecure,no_root_squash,no_subtree_check 10.0.0.10:/ /mnt/nfs
sudo mount -t nfs -o defaults,nfsvers=4.2,rw,intr,nofail,nodiratime,noatime,async,x-systemd.automount,_netdev,rdma,port=20049 10.0.0.10:/home /mnt/nfs

# Check nfs version
mount | grep nfs
sudo umount /mnt/nfs

sudo rm -R /mnt/nfs
```

### Mont options explained
Mount Options
- `nfsvers=4.2` - Use NFSv4.2
- `defaults` - Use default options: rw, suid, dev, exec, auto, nouser, async
- `rw` - Read and write
- `intr` - Allow NFS operations to be interrupted
- `nofail` - Do not fail when the server is not available
- `nodiratime` - Do not update directory access time
- `noatime` - Do not update file access time
- `async` - Write data to the disk asynchronously
- `x-systemd.automount` - Automatically mount the NFS share
- `_netdev` - Wait for the network to be available
- `rdma` - Use RDMA protocol
- `port=20049` - Use port 20049
- `fsc` - For cashing (needs cachefilesd)

Mount Path explained for NFSv4 (difference to NFSv3)
- `fsid=0` - Export the root directory
- `insecure` - Allow connections from non-reserved ports
- `no_root_squash` - Do not map root user to nobody
- `no_subtree_check` - Do not check the subtree
- `anonuid=65534` - Map the anonymous user to nobody
- `anongid=65534` - Map the anonymous group to nobody

### Create fstab entries
```bash	
# Install dependencies
sudo apt update
sudo apt install -y nfs-common cachefilesd

# Create Folder
sudo mkdir -p /mnt/nfs
sudo mkdir -p /mnt/rdma_nfs

sudo chown -R nobody:nogroup /mnt/nfs
sudo chmod -R 777 /mnt/nfs

sudo chown -R nobody:nogroup /mnt/rdma_nfs
sudo chmod -R 777 /mnt/rdma_nfs

sudo mount -t nfs -o nfsvers=4.2,defaults,rw,intr,nofail,nodiratime,noatime,async,x-systemd.automount,_netdev 10.0.0.10:/ /mnt/nfs
sudo umount /mnt/nfs

sudo nano /etc/fstab
# Mount user specific folder (with RDMA)
10.0.0.10:/exports/home/<user> /mnt/rdma_nfs nfs defaults,nfsvers=4.2,fcs,intr,nofail,nodiratime,noatime,x-systemd.automount,_netdev,rdma,port=20049 0 0

sudo systemctl daemon-reload
sudo sudo mount -a
```

## Performance Tests
- Install performance test tools
```bash	
sudo apt update
sudo apt install fio hdparm sysstat bonnie++ iftop iotop iozone3 nfs-common iperf ioping nfs-common
```

### Hard Drive Performance
- Get IO Statistic
```bash
iostat
# or continuous
iostat -m 2
```

- Test maximum hard drive speed
```bash	
sudo hdparm -Tt /dev/nvme-raid/raid
iostat -mx 2 /dev/nvme-raid/raid
```

- Benchmark the raid 
```bash	
# fio benchmark
raid_path=/mnt/raid/raidtest
mkdir -p $raid_path
# randrw, read, write, randread, randwrite
fio \
	--rw=randread \
	--bs=4M \
	--numjobs=1 \
	--iodepth=1 \
	--runtime=30 \
	--time_based \
	--loops=1 \
	--ioengine=libaio \
	--direct=0 \
	--invalidate=1 \
	--fsync_on_close=1 \
	--randrepeat=1 \
	--norandommap \
	--exitall \
	--filesize=1G \
	--group_reporting \
    --refill_buffers \
    --name raidtest \
    --directory=$raid_path \
    --nrfiles=16

# Get sequential write and read speed (single file and core)
dd if=/dev/zero of=$raid_path/raidtest.txt bs=1024M count=256 status=progress oflag=direct
dd if=$raid_path/raidtest.txt of=/dev/null bs=4M status=progress iflag=direct

dd if=/dev/zero of=$raid_path/raidtest.txt bs=1024M count=256 status=progress
dd if=$raid_path/raidtest.txt of=/dev/null bs=4M status=progress

# Get latency
ioping -c 10 $raid_path

# Takes long as it needs double size of RAM
sudo bonnie++ -d $raid_path -u root -m $raid_path 
# Save last line of bonnie++ output to csv
bon_csv2html benchmark/nfs.csv > benchmark/nfs.html
```

- Test network speed
```bash
# On Server
iperf -s

# On Client
iperf -c 10.0.0.10 -f m -t 60
```

- Test normal NFS Speed
```bash
raid_path=/mnt/rdma_nfs/raidtest

# Get latency
ioping -c 10 $raid_path

# -a - All tests
# -g 1G - File size
# -i 0 - Write test
# -i 1 - Read test
# -i 2 - Random read/write test
# -f $raid_path/iozone_test - File path
# -t 2 - Number of threads
# -I - Disable caching


# Performance test
iozone -a -g 1G -i 0 -i 1 -i 2 -f $raid_path
iozone -g 1G -i 0 -i 1 -i 2 -f $raid_path
sudo nfsiostat 1 10 $raid_path
```

- Test RDMA NFS Speed
```bash
# Test on Client

# Create new mount or use the one from fstab
sudo mkdir -p /mnt/rdma_test 
sudo mount -t nfs4 -o rdma,port=20049 10.0.0.10:/ /mnt/rdma_test 
mount | grep proto=rdma

raid_path=/mnt/rdma_nfs/raidtest

mkdir -p $raid_path

# randrw, read, write, randread, randwrite
fio \
	--rw=randread \
	--bs=4M \
	--numjobs=1 \
	--iodepth=1 \
	--runtime=30 \
	--time_based \
	--loops=1 \
	--ioengine=libaio \
	--direct=0 \
	--invalidate=1 \
	--fsync_on_close=1 \
	--randrepeat=1 \
	--norandommap \
	--exitall \
	--filesize=1G \
	--group_reporting \
    --refill_buffers \
    --name raidtest \
    --directory=$raid_path \
    --nrfiles=16

sudo umount /mnt/rdma_test 
sudo rm -R /mnt/rdma_test 
```