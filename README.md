# Install-Lustre2.12.8-on-CentOS-7.9-2009 with ZFS 0.7.13


I tried to compile and install Lustre 2.12.8 on the CentOS 7.9, with success and also failures.

It seems the the KMOD compiling on the CentOS 7.9 has alot of issue on the kmod-lustre-osd-zfs. The compiling for  "make rpms" can be fine with 
some changes to the scripts, while it still comes with a lot of kernel ksym errors when install using yum install. 

```text
Error: Package: kmod-lustre-osd-zfs-2.12.8_6_g5457c37-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.8_6_g5457c37-1.el7.x86_64)
           Requires: ksym(dmu_tx_hold_sa) = 0x81757757
Error: Package: kmod-lustre-osd-zfs-2.12.8_6_g5457c37-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.8_6_g5457c37-1.el7.x86_64)
           Requires: ksym(dmu_object_free) = 0x0b33c89f
Error: Package: kmod-lustre-osd-zfs-2.12.8_6_g5457c37-1.el7.x86_64 (/kmod-lustre-osd-zfs-2.12.8_6_g5457c37-1.el7.x86_64)

```

By using "rpm -ivh --nodeps" can 
force the compiled kmod rpms to install, and then the mkfs.lustre works fine with the ZFS pools look OK. But then when trying to mount the pools, it will 
report error and failed with saying something like "the drives are not formated by mkfs.lustre, or the backend filesystem is not supported".

I have tried different sub versions of 3.10.0-1160 kernels, Lustre patched kernel,  even downgraded the rpm/rpm-build packages, with the same error.


I finnaly decide to go with DKMS modules for all the ZFS/SPL, and Lustre, which turned out working fine.

1. Download the rpms from https://downloads.whamcloud.com/public/lustre/lustre-2.12.8/el7.9.2009/server/RPMS/x86_64/

```text

zfs-dkms-0.7.13-1.el7.noarch.rpm  lustre-zfs-dkms-2.12.8_6_g5457c37-1.el7.noarch.rpm       libzfs2-0.7.13-1.el7.x86_64.rpm
zfs-0.7.13-1.el7.x86_64.rpm       lustre-osd-zfs-mount-2.12.8_6_g5457c37-1.el7.x86_64.rpm  libuutil1-0.7.13-1.el7.x86_64.rpm
spl-dkms-0.7.13-1.el7.noarch.rpm  lustre-2.12.8_6_g5457c37-1.el7.x86_64.rpm                libnvpair1-0.7.13-1.el7.x86_64.rpm
spl-0.7.13-1.el7.x86_64.rpm       libzpool2-0.7.13-1.el7.x86_64.rpm

```

2. Install all the above rpm pacjages:

```text
yum localinstall spl*.rpm
yum localinstall zfs*.rpm lib*rpm
```
The above installation work smoothly. You can run "dkms status" to check.

One issue here is that it will report that the feature of "REMAKE_INITRD" is deprecated. To avoid this and also let the following Lustre installation 
work fine, I need to comment one line on each of the "dkms.cof" file of ZFS and SPL.
```text
vi /usr/src/spl-0.7.13/dkms.conf

...

#REMAKE_INITRD="no


vi /usr/src/zfs-0.7.13/dkms.conf

#REMAKE_INITRD="no

```
Now the "dkms status" should list something like:

```text

dkms status
spl/0.7.13, 3.10.0-1160.71.1.el7.x86_64, x86_64: installed
zfs/0.7.13, 3.10.0-1160.71.1.el7.x86_64, x86_64: installed (original_module exists)

```

3. Now install the Lustre
```text
yum localinstall lustre*.rpm
 ```
 
 The installaton may looks fine, wile it reports error for compiling the Lustre modules. The DKMS source and the Lustre utilities have 
 actually been installed, but the modules are not compiled and installed sucessfully. Run "modprob lustre" will simply reporting eror.
 
 The issue looks like is caused by a) the configure scipt can not properly find the ZFS source; b) the "dkms" has changed it's way to show the modules' status.
 ```text
 OLD:
 dkms status
spl, 0.7.13, 3.10.0-1160.71.1.el7.x86_64, x86_64: installed
 
 
 NEW:
 dkms status
spl/0.7.13, 3.10.0-1160.71.1.el7.x86_64, x86_64: installed
 
 ```
 The change of "," to "/" of showing the modules' names causes this trouble here.
 
 There are 3 changes on 3 files needed to be able to build and install the DKMS modules sucessfully:
 ```text
 a)
 vi /usr/src/lustre-zfs-2.12.8_6_g5457c37/dkms.conf

#REMAKE_INITRD="no



b)

vi configure

change line 33099 from:

zfsver=$(ls -1 /usr/src/ | grep -m1 zfs | cut -f2 -d'-')

to 

zfsver=$(ls -1 /usr/src/ | grep -m1 ^zfs | cut -f2 -d'-')

#Note the "^" added.

c)
vi lustre-dkms_pre-build.sh

change line 23 from:

ZFS_VERSION=$(dkms status -m zfs -k $3 -a $5 | awk -F', ' '{print $2; exit 0}' | grep -v ': added$')

to

ZFS_VERSION=$(dkms status -m zfs -k $3 -a $5 | awk -F', ' '{print $1; exit 0}' | cut -f2 -d'/'| cut -f1 -d':')


```
 
 After this, run:
 
 ```text
 
 dkms build lustre-zfs/2.12.8_6_g5457c37
 
 dkms install lustre-zfs/2.12.8_6_g5457c37
 
 modprob lustre
 
 ```
 
 Once it's done, it should fine.
 
 4. Test on configuring Lustre storage.
 
 ```text
 
for var in 21 22; do dd if=/dev/zero of=/opt/hd${var}.img bs=1024K count=5120; done
 
losetup -o 1048576 /dev/loop21 /opt/hd21.img;losetup -o 1048576 /dev/loop22 /opt/hd22.img
 
mkfs.lustre --reformat --mdt --mgs --backfstype=zfs --fsname=mylustre --mgsnode=192.168.0.1 --index=0 zfspool/mds /dev/loop21
mkfs.lustre --reformat --ost --backfstype=zfs --fsname=mylustre --mgsnode=192.168.0.1 --index=1 zfspoolost/ost /dev/loop22
 
mkdir -p /mnt/lustre/mylustre-mdt
mkdir -p /mnt/lustre/mylustre-ost

mount -t lustre zfspool/mds /mnt/lustre/mylustre-mdt/
mount -t lustre zfspoolost/ost /mnt/lustre/mylustre-ost/

mount.lustre 192.168.0.1@tcp:/mylustre /lustre_store
 ```
 
 Since the dkms Lustre modules also include the client's module, so we can directly mount the newly set Lustre storage. 
 
 For he server side, it seems to me that only the osd-zfs module(and maybe some related modules) needed.
 
 
 ```text
 df -h
Filesystem                   Size  Used Avail Use% Mounted on
devtmpfs                      24G     0   24G   0% /dev
...
zfspool/mds                  4.8G  3.2M  4.8G   1% /mnt/lustre/mylustre-mdt
zfspoolost/ost               4.8G  3.0M  4.8G   1% /mnt/lustre/mylustre-ost
192.168.0.1@tcp:/mylustre    4.8G  3.0M  4.8G   1% /lustre_store
 
 ```
 
