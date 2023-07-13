
### VirtualBox Dynamic Disk Not Utilizing Available Free Disk Space to Grow Partition Automatically

**Problem Statement:** "Although a 32GB dynamic disk was created, the OS partition hit a `disk full` message at around `4GB` and wasn't able to automatically increase its size by utilizing the `free` 28GB from the 32GB dynamic disk."

If this is the problem you encountered, and you had setup the Guest OS using Ubuntu Linux and LVM (and EXT4 filesystem), then the solution to this problem is to use `lvextend` and `resize2fs` commands. See my solution and description below:

Start up the VM and login. Once in, open up a terminal.

Display information about allocated disk storage:


`$ df -h`

```
Filesystem Size Used Avail Use% Mounted on
udev 966M 0 966M 0% /dev
tmpfs 200M 1.1M 199M 1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv 3.9G 2.9G 835M 78% /
tmpfs 997M 0 997M 0% /dev/shm
tmpfs 5.0M 0 5.0M 0% /run/lock
tmpfs 997M 0 997M 0% /sys/fs/cgroup
/dev/loop0 97M 97M 0 100% /snap/core/9436
/dev/loop1 97M 97M 0 100% /snap/core/9665
/dev/sda2 976M 212M 697M 24% /boot
tmpfs 200M 0 200M 0% /run/user/1000

```

The output shows the root partition `/` has a size of 3.9G and that 2.9G (78%) is used.

Display information on block devices:


`$ lsblk`

```
NAME MAJ:MIN RM SIZE RO TYPE MOUNTPOINT
loop0 7:0 0 96.5M 1 loop /snap/core/9436
loop1 7:1 0 97M 1 loop /snap/core/9665
sda 8:0 0 32G 0 disk
├─sda1 8:1 0 1M 0 part
├─sda2 8:2 0 1G 0 part /boot
└─sda3 8:3 0 31G 0 part
└─ubuntu--vg-ubuntu--lv 253:0 0 4G 0 lvm /
sr0 11:0 1 1024M 0 rom

```
The output shows that block device sda has a physical size of 32GB. It contains 3 partitions (sda1, sda2 and sda3). The root partition (/) mounted on the logical volume of sda3 only has 4GB allocated storage. This means that there is still approximately 27GB of unallocated (free) storage that we can use to grow the LVM's logical volume.

Display filesystem information on the block devices:

`$ lsblk --fs`

```
NAME FSTYPE LABEL UUID MOUNTPOINT
loop0 squashfs /snap/core/9436
loop1 squashfs /snap/core/9665
sda
├─sda1
├─sda2 ext4 7cafee4a-d300-4b17-957d-c822c73882fc /boot
└─sda3 LVM2_member GwPZus-EMpD-BgZw-e1Nx-Fd9J-LLeN-XnbvEE └─ubuntu--vg-ubuntu--lv ext4 dcc015b4-b6b8-4e05-9a18-93c263cc05ed /
sr0

```
The output shows that filesystem of the lvm root partition is ext4.

Display information about physical volume:

`$ sudo pvs`

```
PV VG Fmt Attr PSize PFree
/dev/sda3 ubuntu-vg lvm2 a-- <31.00g <27.00g

```
`$ sudo pvdisplay`

```
--- Physical volume --- PV Name /dev/sda3
VG Name ubuntu-vg
PV Size <31.00 GiB / not usable 0
Allocatable yes
PE Size 4.00 MiB
Total PE 7935
Free PE 6911
Allocated PE 1024
PV UUID GwPZus-EMpD-BgZw-e1Nx-Fd9J-LLeN-XnbvEE
```

The output provides another perspective on the allocated / unallocated space on LVM PV sda3.

Display information about volume group:

`$ sudo vgs`

```
VG #PV #LV #SN Attr VSize VFree
ubuntu-vg 1 1 0 wz--n- <31.00g <27.00g
```
`$ sudo vgdisplay`

```
--- Volume group --- VG Name ubuntu-vg
System ID
Format lvm2
Metadata Areas 1
Metadata Sequence No 2
VG Access read/write
VG Status resizable
MAX LV 0
Cur LV 1
Open LV 1
Max PV 0
Cur PV 1
Act PV 1
VG Size <31.00 GiB
PE Size 4.00 MiB
Total PE 7935
Alloc PE / Size 1024 / 4.00 GiB
Free PE / Size 6911 / <27.00 GiB
VG UUID vBmSeb-VWR8-7A8n-CBSa-UnKd-FmJG-LTtMUG

```

The output shows that the VG size is the same as the PV size (31GB), and that it has 27GB of unallocated free storage to grow the logical volume.

Display information about the logical volume:

`$ sudo lvs`

```
LV VG Attr LSize Pool Origin Data% Meta% Move Log Cpy%Sync Convert
ubuntu-lv ubuntu-vg -wi-ao---- 4.00g
```

`$ sudo lvdisplay`

```
--- Logical volume ---
LV Path /dev/ubuntu-vg/ubuntu-lv
LV Name ubuntu-lv
VG Name ubuntu-vg
LV UUID pnvBcu-FnfT-c3RY-cYCE-AuMX-f833-57WvYi
LV Write Access read/write
LV Creation host, time ubuntu-server, 2020-07-01 00:00:58 +0000
LV Status available
#open 1
LV Size 4.00 GiB
Current LE 1024
Segments 1
Allocation inherit
Read ahead sectors auto
-currently set to 256
Block device 253:0
```

The output shows that the LV size is 4GB. Since the VG size is 31GB, we can grow the LV by another 27GB.

To grow the logical volume (ubuntu-lv) by using 50% (13GB) of the free unallocated space, first do:

`$ sudo lvextend -l+50%free /dev/ubuntu-vg/ubuntu-lv`

```
Size of logical volume ubuntu-vg/ubuntu-lv changed from 4.00 GiB (1024 extents) to 17.50 GiB (4480 extents).
Logical volume ubuntu-vg/ubuntu-lv successfully resized.
```

The output shows that the logical volume has grown from 4GB to 17.5GB (approximately 13GB).
7b. Finally, to let the EXT4 filesystem become aware of the expanded storage, run command:

`$ sudo resize2fs /dev/ubuntu-vg/ubuntu-lv`

```
resize2fs 1.44.1 (24-Mar-2018)
Filesystem at /dev/ubuntu-vg/ubuntu-lv is mounted on /; on-line resizing required
old_desc_blocks = 1, new_desc_blocks = 3
The filesystem on /dev/ubuntu-vg/ubuntu-lv is now 4587520 (4k) blocks long.

```
Display information about allocated disk storage again to check the new disk storage space:

`$ df -h`

```
Filesystem Size Used Avail Use% Mounted on
udev 966M 0 966M 0% /dev
tmpfs 200M 1.1M 199M 1% /run
/dev/mapper/ubuntu--vg-ubuntu--lv 18G 2.9G 14G 18% /
tmpfs 997M 0 997M 0% /dev/shm
tmpfs 5.0M 0 5.0M 0% /run/lock
tmpfs 997M 0 997M 0% /sys/fs/cgroup
/dev/loop0 97M 97M 0 100% /snap/core/9436
/dev/loop1 97M 97M 0 100% /snap/core/9665
/dev/sda2 976M 212M 697M 24% /boot
tmpfs 200M 0 200M 0% /run/user/1000
```

The output shows that the size of the logical volume has now grown to 18GB (from its earlier 4GB)
