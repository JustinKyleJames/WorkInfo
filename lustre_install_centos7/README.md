# Setup Base Machine

This is a combination of:
https://gist.github.com/Jorgeelalto/71798d4c3636187e66e9979946dad424
https://github.com/lrahmani/lustre-on-virtualbox


1. Boot the vm with CentOS-7-x86_64-Minimal-1708.iso as a live CD
2. Configure adapter 1 as Bridged Adapter
3. Configure adapter 2 as NAT.  (Enable port forwarding of port 988 guest <-> host.
4. Set ONBOOT=yes in /etc/sysconfig/network-scripts/ifcfg-enp0s3 and ifcfg-enp0s8.
5. Install the following packages.

```
## dependencies from http://lustre.ornl.gov/lustre101-courses/content/C1/L3/LustreBasicInstall.pdf, page 10
yum -y install perl libgssglue net-snmp libyaml sg3_utils openmpi lsof rsync
## useful tools
yum -y install wget net-tools telnet vim
```

5. Configure the system:

```
## Ensure that each machine has a non-loopback entry for itself in /etc/hosts
echo "" > /etc/hosts
## Create an entry in /etc/modprobe.d/lustre.conf
echo "options lnet networks=tcp(enp0s3)" > /etc/modprobe.d/lustre.conf
## Disable firewall
systemctl disable firewalld
systemctl stop firewalld
## Disable SELinux
sed -i '/^SELINUX=/c\SELINUX=disabled' /etc/sysconfig/selinux
sed -i '/^SELINUX=/c\SELINUX=disabled' /etc/selinux/config
```

6. Stop the system and clone it three times as MDS, OSS, and client.

# Install software on MDS/MGS and OSS.

1. Get the RPM packages.

```
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/kernel-3.10.0-693.5.2.el7_lustre.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/lustre-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/lustre-iokit-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/lustre-osd-ldiskfs-mount-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/lustre-tests-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/kmod-lustre-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/kmod-lustre-osd-ldiskfs-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/server/RPMS/x86_64/kmod-lustre-tests-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/e2fsprogs/1.42.13.wc6/el7/RPMS/x86_64/e2fsprogs-1.42.13.wc6-7.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/e2fsprogs/1.42.13.wc6/el7/RPMS/x86_64/e2fsprogs-libs-1.42.13.wc6-7.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/e2fsprogs/1.42.13.wc6/el7/RPMS/x86_64/libcom_err-1.42.13.wc6-7.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/e2fsprogs/1.42.13.wc6/el7/RPMS/x86_64/libss-1.42.13.wc6-7.el7.x86_64.rpm
```

2. Install Lustre kernel RPM.

```
yum localinstall kernel-3.10.0-693.5.2.el7_lustre.x86_64.rpm
```
3. Replace the kernel with the new one.

```
/sbin/new-kernel-pkg --package kernel --mkinitrd --dracut --depmod --install 3.10.0-693.5.2.el7_lustre.x86_64
shutdown -r now
```

4. Install e2fsprogs (Provides utilities for working with ext2/ext3/ext4 file systems).

```
yum localinstall e2fsprogs-1.42.13.wc6-7.el7.x86_64.rpm  e2fsprogs-libs-1.42.13.wc6-7.el7.x86_64.rpm \
libcom_err-1.42.13.wc6-7.el7.x86_64.rpm libss-1.42.13.wc6-7.el7.x86_64.rpm
```

5. Install Lustre modules.

```
yum localinstall kmod-lustre-2.10.2-1.el7.x86_64.rpm kmod-lustre-osd-ldiskfs-2.10.2-1.el7.x86_64.rpm \
lustre-2.10.2-1.el7.x86_64.rpm lustre-osd-ldiskfs-mount-2.10.2-1.el7.x86_64.rpm \
lustre-iokit-2.10.2-1.el7.x86_64.rpm
```

6. Install Lustre test modules.

```
yum localinstall kmod-lustre-tests-2.10.2-1.el7.x86_64.rpm lustre-tests-2.10.2-1.el7.x86_64.rpm
```

7. Configure LNET

```
lnetctl net add --net tcp0 --if enp0s3
```

# Configure MDS/MGS

1. Stop MDS VM.

2. Create two hard drives under Storage->Controller:SATA.

3. Start MDS VM.

4.  Format drives.

```
mkfs.lustre --mgs /dev/sdb
mkfs.lustre --reformat --fsname=lustre --mgsnode=<MGS IP>@tcp --mdt --index=0  /dev/sdc
```

5.  Start MGS/MDS.

```
mkdir /mnt/mgt /mnt/mdt
mount -t lustre /dev/sdb /mnt/mgt # MGS
mount -t lustre /dev/sdc /mnt/mdt # MDS
```

6.  Update changelog mask and register a listener.

```
sudo lctl set_param mdd.lustre-MDT0000.changelog_mask="CREAT CLOSE RENME UNLNK MKDIR RMDIR"
lctl --device lustre-MDT0000 changelog_register
```

# Configure OSS

1.  Stop OSS VM.

2.  Create a hard drive under Storage->Controller:SATA.

3.  Start OSS VM.

4.  Format drive.

```
mkfs.lustre --reformat --fsname=lustre --ost --mgsnode=<MGS IP>@tcp --index=2 /dev/sdb
```

5.  Start OSS.

```
mkdir /mnt/ost2
mount -t lustre /dev/sdb /mnt/ost2/
```

# Install and Configure Client

1. Download client packages.

```
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/client/RPMS/x86_64/kmod-lustre-client-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/client/RPMS/x86_64/lustre-client-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/client/RPMS/x86_64/kmod-lustre-client-tests-2.10.2-1.el7.x86_64.rpm
wget https://downloads.whamcloud.com/public/lustre/lustre-2.10.2/el7/client/RPMS/x86_64/lustre-client-tests-2.10.2-1.el7.x86_64.rpm
```

2. Install client RPMs.

```
yum localinstall kmod-lustre-client-2.10.2-1.el7.x86_64.rpm lustre-client-2.10.2-1.el7.x86_64.rpm
```

Note:  Installation of test RPMs failed but not necessary.

3. Make a directory for a Lustre mount.

```
mkdir /mnt/lustre
```

4. Mount Lustre.

```
mount -t lustre <MGS IP>@tcp:/lustre /mnt/lustre/
```

5.  Test it out.

```
echo 'this is a test' > /mnt/lustre/test.txt
```

6.  Read changelog:

```
lfs changelog lustre-MDT0000
```


