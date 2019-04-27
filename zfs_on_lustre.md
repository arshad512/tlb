# 110
# Ubuntu-18.04 (bionic). Kernel 4.9. ZFS on Single node client/server Lustre (HEAD).
### Arshad Hussain(arshad.super@gmail.com)

### Get libs and dev packages
These libs and devel packages are required. Better to get them upfront.
```
$ apt-get install libssl-dev libelf-dev libmount-dev libext2fs-dev uuid-dev libblkid-dev dietlibc-dev
$ apt-get install build-essential debhelper devscripts fakeroot kernel-wedge libudev-dev pciutils-dev
$ apt-get install texinfo xmlto libelf-dev python-dev liblzma-dev libaudit-dev dh-systemd libyaml-dev
$ apt-get install module-assistant libreadline-dev dpatch libsnmp-dev quilt
```

### Get and compile kernel (4.9.170)
This takes 3.5 hours to compile. This requires 30 gigs of space.
```
KPATH=/root/lk_4.6

$ mkdir -p $KPATH
$ cd $KPATH
$ wget https://cdn.kernel.org/pub/linux/kernel/v4.x/linux-4.9.170.tar.xz
$ yes '' | make oldconfig
$ make clean
$ make -j2 deb-pkg
$ dpkg -i linux-image-4.9.170_4.9.170-1_amd64.deb
$ dpkg -i linux-headers-4.9.170_4.9.170-1_amd64.deb
$ reboot
```

### Get and build SPL (Solaris porting layer)
```
KPATH=/root/lk_4.6
SPLPATH=/root/zfs/spl

$ mkdir -p $SPLPATH
$ cd $SPLPATH
$ wget http://archive.ubuntu.com/ubuntu/pool/universe/s/spl-linux/spl-linux_0.7.12.orig.tar.gz
$ ./configure --with-linux=$KPATH/linux-4.9.170
$ make -j2
$ make install
```

### Get and build ZFS
```
KPATH=/root/lk_4.6
SPLPATH=/root/zfs/spl
ZFSPATH=/root/zfs/zfs

$ mkdir -p $ZFSPATH
$ cd $ZFSPATH
$ wget http://archive.ubuntu.com/ubuntu/pool/main/z/zfs-linux/zfs-linux_0.7.12.orig.tar.gz
$ ./configure --with-linux=$KPATH/linux-4.9.170 --with-spl=$SPLPATH/spl-0.7.12/
$ make -j2
$ make install
```

### Re-creates list of module dependencies
Not requried, however, rebooting is adviced here.
```
$ depmod -a
$ reboot
```

### Build Lustre with ZFS
```
KPATH=/root/lk_4.6
SPLPATH=/root/zfs/spl
ZFSPATH=/root/zfs/zfs

$ git clone git://git.whamcloud.com/fs/lustre-release.git
$ sh ./autogen.sh
$ ./configure --disable-ldiskfs --with-linux=$KPATH/linux-4.9.170 --with-zfs=$ZFSPATH/zfs-0.7.12 --with-spl=$SPLPATH/spl-0.7.12/
$ make -j2
$ make install
```

### Setup network and hostname
```
TODO
```

### Setup Lustre network module to load at startup
```
root@lhack:~# echo "options lnet networks=tcp0(enp0s8)" >>  /etc/modprobe.d/lustre.conf
root@lhack:~# cat /etc/modprobe.d/lustre.conf
options lnet networks=tcp0(enp0s8)
root@lhack:~#
```
### Verify NID
```
root@lhack:~# modprobe lnet
root@lhack:~# lctl network up
LNET configured
root@lhack:~# lctl list_nids
10.106.9.50@tcp
root@lhack:~#
```

### Load modules
If these work, we are almost done.
```
$ modprobe lnet
$ modprobe zfs
$ modporbe lustre
```

### Format and Mount MGS/MDS.
/dev/sdb1 is used as device for MGS/MDS.
```
$ mkdir -p /mnt/mdt
$ mkfs.lustre --reformat --backfstype=zfs --fsname=lustre --mgsnode=10.106.9.50@tcp --mgs --mdt gpool/metadata /dev/sdb1 
$ mount.lustre gpool/metadata /mnt/mdt
```

### Format and Mount OST.
/dev/sdb2 is used as device OSS/OST
```
$ mkdir -p /mnt/ost
$ mkfs.lustre --index 0 --mgsnode=10.106.9.50@tcp --backfstype=zfs --fsname=lustre --ost gpool/data /dev/sdb2
$ mount.lustre gpool/data /mnt/ost/
```

### Verify Zpool(gpool) which have been created.
```
root@lhack:~/lustre_dev/lustre-release# zpool list
NAME    SIZE  ALLOC   FREE  EXPANDSZ   FRAG    CAP  DEDUP  HEALTH  ALTROOT
gpool  2.98G  6.35M  2.98G         -     0%     0%  1.00x  ONLINE  -
root@lhack:~/lustre_dev/lustre-release#

root@lhack:~/lustre_dev/lustre-release# zfs list
NAME             USED  AVAIL  REFER  MOUNTPOINT
gpool           6.33M  2.85G    24K  /gpool
gpool/data      3.11M  2.85G  3.11M  /gpool/data
gpool/metadata  3.05M  2.85G  3.05M  /gpool/metadata
root@lhack:~/lustre_dev/lustre-release#
```

### Finally mount the Client
```
$ mkdir -p /mnt/lustre
$ mount -t lustre 10.106.9.50@tcp:/lustre /mnt/lustre/
```

## Time to play around with Lustre
```
root@lhack:~/lustre_dev/lustre-release# dd if=/dev/zero of=/mnt/lustre/test_file bs=512 count=100
100+0 records in
100+0 records out
51200 bytes (51 kB, 50 KiB) copied, 0.00113225 s, 45.2 MB/s
root@lhack:~/lustre_dev/lustre-release#
```

```
root@lhack:~/lustre_dev/lustre-release# ls -ali /mnt/lustre/test_file 
144115305935798273 -rw-r--r-- 1 root root 51200 Apr 26 11:27 /mnt/lustre/test_file
root@lhack:~/lustre_dev/lustre-release#
```
```
root@lhack:~/lustre_dev/lustre-release# stat /mnt/lustre/test_file
 File: /mnt/lustre/test_file
 Size: 51200           Blocks: 1          IO Block: 4194304 regular file
Device: 2c54f966h/743766374d    Inode: 144115305935798273  Links: 1
Access: (0644/-rw-r--r--)  Uid: (    0/    root)   Gid: (    0/    root)
Access: 2019-04-26 11:27:00.000000000 +0000
Modify: 2019-04-26 11:27:00.000000000 +0000
Change::0
 2019-04-26 11:27:00.000000000 +0000
Birth: -
root@lhack:~/lustre_dev/lustre-release#
```
```
root@lhack:~/lustre_dev/lustre-release# lfs path2fid /mnt/lustre/test_file
[0x200001b71:0x1:0x0]
root@lhack:~/lustre_dev/lustre-release#
```
```
root@lhack:~/lustre_dev/lustre-release# lfs fid2path lustre 0x200001b71:0x1:0x0
test_file
root@lhack:~/lustre_dev/lustre-release#
```
```
root@lhack:~/lustre_dev/lustre-release# lfs getstripe /mnt/lustre/test_file
/mnt/lustre/test_file
lmm_stripe_count:  1
lmm_stripe_size:   1048576
lmm_pattern:       raid0
lmm_layout_gen:    0
lmm_stripe_offset: 0
       obdidx           objid           objid           group
            0              98           0x62                0

root@lhack:~/lustre_dev/lustre-release#
```
```
root@lhack:~/lustre_dev/lustre-release# lfs getsom /mnt/lustre/test_file
file: /mnt/lustre/test_file size: 51200 blocks: 1 flags: 4
root@lhack:~/lustre_dev/lustre-release#
```
```
root@lhack:~/lustre_dev/lustre-release# lfs osts
OBDS:
0: lustre-OST0000_UUID ACTIVE
root@lhack:~/lustre_dev/lustre-release#
```
```
root@lhack:~/lustre_dev/lustre-release# lfs mdts
MDTS:
0: lustre-MDT0000_UUID ACTIVE
root@lhack:~/lustre_dev/lustre-release#
```

## Happy Hacking!!!
