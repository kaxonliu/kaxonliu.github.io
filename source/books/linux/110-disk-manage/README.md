# 存储管理

## 硬盘在系统中的名称

普通硬盘在系统中的名称以 `sd` 开头，`sda` 表示第一个硬盘，`sdb` 表示第二个硬盘，以此类推。对一个硬盘做分区，第一个分区的名称是 `sda1`，第二个分区是 `sda2`，以此类推。这些信息在 `/dev/`中可以查看到。

固态硬盘在系统中的名称以 `nvme` 开头，`nvme0` 表示第一个控制编号，`nvme0n1` 表示上面的第一个硬盘，`nvme0n1p1` 表示第一个分区。这些信息也是在 `/dev`中查看

~~~bash
root@ubuntu:/dev# ll nvm*
crw------- 1 root root 237, 0 Aug 17  2025 nvme0
brw-rw---- 1 root disk 259, 0 Aug 17  2025 nvme0n1
brw-rw---- 1 root disk 259, 1 Aug 17  2025 nvme0n1p1
brw-rw---- 1 root disk 259, 2 Aug 17  2025 nvme0n1p2
brw-rw---- 1 root disk 259, 3 Aug 17  2025 nvme0n1p3
~~~

**增加新硬盘**。新增硬盘后重启系统，如果需要无法加载系统的情况，这时候需要进入 bios，找到启动盘，把启动盘的优先级调上去，然后保存。





## 分区的工具和方法

**硬盘分区格式**

- MBR 的意思是 "主引导记录"。MBR 最大支持 2TB 容量，在容量方面存在着极大的瓶颈。并且只支持创建最多4个主分区，。而GPT分区方式就没有这些限制。
- GPT 分区（ubuntu装系统默认就是GPT分区），GPT 意为 GUID 分区表，它支持的磁盘容量比 MBR 大得多。这是一个正逐渐取代 MBR 的新标准，它是由 UEFI 辅住而形成的，将来 UEFI 用于取代老旧的 BIOS，而 GPT 则取代老旧的 MBR。

**硬盘分区工具**

- `fdisk` 工具适用于 MBR 格式
- `gdisk` 工具适用于 GPT 格式



使用 `lsblk` 命令 查看硬盘的分区情况。可以看到硬盘 `nvme0n2` 已经分区了，但是硬盘 `nvme0n1` 未分区。

~~~bash
[root@rocky ~]# lsblk
NAME        MAJ:MIN RM  SIZE RO TYPE MOUNTPOINTS
sr0          11:0    1  1.9G  0 rom  /opt
nvme0n1     259:0    0   20G  0 disk
nvme0n2     259:1    0   20G  0 disk
├─nvme0n2p1 259:2    0  500M  0 part /boot/efi
├─nvme0n2p2 259:3    0    2G  0 part [SWAP]
└─nvme0n2p3 259:4    0 17.5G  0 part /
~~~



## fdisk分区

`fdisk` 工具适用于空间小于 2TB的硬盘，分区类型为MBR，主分区4或者主分区3+扩展分区（包含逻辑分区）`fdisk` 分区时可以使用下面的命令，分区过程是在内存中完成的，需要最后使用 `w` 保存后才能生效。

~~~bash
# 常用的4个命令
n add a new partition     #新建分区
p print the partition table     #显示分区表的信息
d delete a partition     #删除分区
w write table to disk and exit     #保存退出

# 其他的命令
m print this menu     #显示帮助菜单
a toggle a bootable flag     #切换分区启动标记
b edit bsd disklabel     #编辑sdb磁盘标签
c toggle the dos compatibility flag     #切换dos兼容模式
l list known partition types     #显示分区类型
o create a new empty DOS partition table #创建新的空白分区表
q quit without saving changes     #不保存退出
s create a new empty Sun disklabel     #创建新的Sun磁盘标签
t change a partitions system id         #修改分区ID,可以通过l查看id
u change display/entry units     #修改容量单位,磁柱或扇区
v verify the partition table     #检验分区表
x extra functionality (experts only)     #拓展功能
~~~



### 分区过程

使用 `fdisk` 给硬盘 `/dev/nvme0n1` 分区。分成3个主分区，一个扩展分区，一个逻辑分区。

~~~bash
[root@rocky ~]# fdisk /dev/nvme0n1

Welcome to fdisk (util-linux 2.37.4).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.

Device does not contain a recognized partition table.
Created a new DOS disklabel with disk identifier 0xed43b6e8.

Command (m for help): n
Partition type
   p   primary (0 primary, 0 extended, 4 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (1-4, default 1):
First sector (2048-41943039, default 2048):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943039, default 41943039): 1g
Value out of range.
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-41943039, default 41943039): +1g

Created a new partition 1 of type 'Linux' and of size 1 GiB.

Command (m for help): p
Disk /dev/nvme0n1: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xed43b6e8

Device         Boot Start     End Sectors Size Id Type
/dev/nvme0n1p1       2048 2099199 2097152   1G 83 Linux

Command (m for help): n
Partition type
   p   primary (1 primary, 0 extended, 3 free)
   e   extended (container for logical partitions)
Select (default p): p
Partition number (2-4, default 2): 2
First sector (2099200-41943039, default 2099200):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2099200-41943039, default 41943039): +1g

Created a new partition 2 of type 'Linux' and of size 1 GiB.

Command (m for help): n
Partition type
   p   primary (2 primary, 0 extended, 2 free)
   e   extended (container for logical partitions)
Select (default p):

Using default response p.
Partition number (3,4, default 3):
First sector (4196352-41943039, default 4196352):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (4196352-41943039, default 41943039): +1g

Created a new partition 3 of type 'Linux' and of size 1 GiB.

Command (m for help): n
Partition type
   p   primary (3 primary, 0 extended, 1 free)
   e   extended (container for logical partitions)
Select (default e):

Using default response e.
Selected partition 4
First sector (6293504-41943039, default 6293504):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (6293504-41943039, default 41943039):

Created a new partition 4 of type 'Extended' and of size 17 GiB.

Command (m for help): p
Disk /dev/nvme0n1: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xed43b6e8

Device         Boot   Start      End  Sectors Size Id Type
/dev/nvme0n1p1         2048  2099199  2097152   1G 83 Linux
/dev/nvme0n1p2      2099200  4196351  2097152   1G 83 Linux
/dev/nvme0n1p3      4196352  6293503  2097152   1G 83 Linux
/dev/nvme0n1p4      6293504 41943039 35649536  17G  5 Extended

Command (m for help): n
All primary partitions are in use.
Adding logical partition 5
First sector (6295552-41943039, default 6295552):
Last sector, +/-sectors or +/-size{K,M,G,T,P} (6295552-41943039, default 41943039): +2g

Created a new partition 5 of type 'Linux' and of size 2 GiB.

Command (m for help): p
Disk /dev/nvme0n1: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xed43b6e8

Device         Boot   Start      End  Sectors Size Id Type
/dev/nvme0n1p1         2048  2099199  2097152   1G 83 Linux
/dev/nvme0n1p2      2099200  4196351  2097152   1G 83 Linux
/dev/nvme0n1p3      4196352  6293503  2097152   1G 83 Linux
/dev/nvme0n1p4      6293504 41943039 35649536  17G  5 Extended
/dev/nvme0n1p5      6295552 10489855  4194304   2G 83 Linux

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
~~~

**使用 `lsblk` 展示分区信息**。分区后可以直接使用（主分区和逻辑分区可以使用）

~~~bash
[root@rocky ~]# lsblk /dev/nvme0n1
NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n1     259:0    0  20G  0 disk
├─nvme0n1p1 259:5    0   1G  0 part
├─nvme0n1p2 259:6    0   1G  0 part
├─nvme0n1p3 259:7    0   1G  0 part
├─nvme0n1p4 259:8    0   1K  0 part
└─nvme0n1p5 259:9    0   2G  0 part
~~~

如何查看哪些分区是主分区和逻辑分区？使用指令 `fdisk -l /dev/nvme0n1`

~~~bash
[root@rocky ~]# fdisk -l /dev/nvme0n1
Disk /dev/nvme0n1: 20 GiB, 21474836480 bytes, 41943040 sectors
Disk model: VMware Virtual NVMe Disk
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xed43b6e8

Device         Boot   Start      End  Sectors Size Id Type
/dev/nvme0n1p1         2048  2099199  2097152   1G 83 Linux
/dev/nvme0n1p2      2099200  4196351  2097152   1G 83 Linux
/dev/nvme0n1p3      4196352  6293503  2097152   1G 83 Linux
/dev/nvme0n1p4      6293504 41943039 35649536  17G  5 Extended
/dev/nvme0n1p5      6295552 10489855  4194304   2G 83 Linux
~~~





## gdisk分区

`gdisk` 工具可以支持大于 2TB的硬盘，分区类型为GPT。`gdisk` 分区使用的命令几乎和 `fdisk`的一样。另外，如果系统上没有 `gdisk` 命令，需要手动安装：`yum install -y gdisk`

### 分区过程

使用 `gdisk` 给硬盘 `/dev/nvme0n3` 分区。分两个区。

~~~bash
[root@rocky ~]# gdisk /dev/nvme0n3
GPT fdisk (gdisk) version 1.0.7

Partition table scan:
  MBR: not present
  BSD: not present
  APM: not present
  GPT: not present

Creating new GPT entries in memory.

Command (? for help): n
Partition number (1-128, default 1):
First sector (34-41943006, default = 2048) or {+-}size{KMGTP}:
Last sector (2048-41943006, default = 41943006) or {+-}size{KMGTP}: +1g
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): p
Disk /dev/nvme0n3: 41943040 sectors, 20.0 GiB
Model: VMware Virtual NVMe Disk
Sector size (logical/physical): 512/512 bytes
Disk identifier (GUID): 7D49EC28-F45C-4CC6-B962-7734ABA06AE1
Partition table holds up to 128 entries
Main partition table begins at sector 2 and ends at sector 33
First usable sector is 34, last usable sector is 41943006
Partitions will be aligned on 2048-sector boundaries
Total free space is 39845821 sectors (19.0 GiB)

Number  Start (sector)    End (sector)  Size       Code  Name
   1            2048         2099199   1024.0 MiB  8300  Linux filesystem

Command (? for help): n
Partition number (2-128, default 2):
First sector (34-41943006, default = 2099200) or {+-}size{KMGTP}:
Last sector (2099200-41943006, default = 41943006) or {+-}size{KMGTP}: +2g
Current type is 8300 (Linux filesystem)
Hex code or GUID (L to show codes, Enter = 8300):
Changed type of partition to 'Linux filesystem'

Command (? for help): w

Final checks complete. About to write GPT data. THIS WILL OVERWRITE EXISTING
PARTITIONS!!

Do you want to proceed? (Y/N):
Your option? (Y/N):
Your option? (Y/N): Y
OK; writing new GUID partition table (GPT) to /dev/nvme0n3.
The operation has completed successfully.

# 查看分区的结果 
[root@rocky ~]# lsblk /dev/nvme0n3
NAME        MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n3     259:10   0  20G  0 disk
├─nvme0n3p1 259:13   0   1G  0 part
└─nvme0n3p2 259:14   0   2G  0 part
~~~



## 制作文件系统

硬盘必须格式化制作文件系统，然后挂载后才能使用。Linux 支持多种文件系统格式，每种格式各有特点，适用于不同场景。
- **ext4 **。属于传统文件系统，是Linux 最常用的默认文件系统，兼容性好，支持日志功能（避免数据损坏），最大支持 **1EB**（1 exabyte）的单文件系统和 **16TB** 的单文件。日志文件系统。
- **XFS**。高性能/现代文件系统，高性能，特别适合大文件（如视频、数据库），支持动态扩容（但不能缩小），最大支持 **8EB** 文件系统。服务器、大数据处理（如 CentOS/RHEL 7+ 的默认文件系统）。日志文件系统。Linux7后默认使用的文件系统。
- **Btrfs (B-tree File System)**。支持写时复制（CoW）、快照、压缩、RAID 集成、子卷管理等功能，但稳定性仍在改进中。需要高级功能（如快照备份）或实验性环境。



### 一块分区的硬盘，指定分区做文件系统

**制作 xfs 文件系统**

~~~bash
[root@rocky ~]# mkfs.xfs /dev/nvme0n1p1
meta-data=/dev/nvme0n1p1         isize=512    agcount=4, agsize=65536 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=262144, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
~~~

**制作 ext4 文件系统**

~~~bash
[root@rocky ~]# mkfs.ext4 /dev/nvme0n3p1
mke2fs 1.46.5 (30-Dec-2021)
Creating filesystem with 262144 4k blocks and 65536 inodes
Filesystem UUID: ad2faeda-20e8-4a99-8ed2-9215b48fba1b
Superblock backups stored on blocks:
	32768, 98304, 163840, 229376

Allocating group tables: done
Writing inode tables: done
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done
~~~



### 一块未分区的硬盘，整个盘做文件系统

~~~bash
[root@rocky ~]# lsblk /dev/nvme0n4
NAME    MAJ:MIN RM SIZE RO TYPE MOUNTPOINTS
nvme0n4 259:13   0   5G  0 disk
[root@rocky ~]#
[root@rocky ~]#
[root@rocky ~]# mkfs.xfs /dev/nvme0n4
meta-data=/dev/nvme0n4           isize=512    agcount=4, agsize=327680 blks
         =                       sectsz=512   attr=2, projid32bit=1
         =                       crc=1        finobt=1, sparse=1, rmapbt=0
         =                       reflink=1    bigtime=1 inobtcount=1 nrext64=0
data     =                       bsize=4096   blocks=1310720, imaxpct=25
         =                       sunit=0      swidth=0 blks
naming   =version 2              bsize=4096   ascii-ci=0, ftype=1
log      =internal log           bsize=4096   blocks=16384, version=2
         =                       sectsz=512   sunit=0 blks, lazy-count=1
realtime =none                   extsz=4096   blocks=0, rtextents=0
~~~





## 挂载和卸载文件系统

上面我们把硬盘 `nvme0n1`和 `nvme0n3`做了分区，`nvme0n4`未做分区。其中，`nvme0n1p1`分区做了 `xfs` 文件系统，`nvme0n3p1` 分区做了 `ext4` 文件系统，`nvme0n4`整个盘做了 `xfs` 文件系统。想要使用他们存放数据就需要挂在到操作系统上。使用 `mount` 挂在到某个文件夹上。

使用 `df` 查看磁盘的挂载情况，`df -h` 友好展示，`df -T` 展示文件系统的类型。

#### 挂载文件系统

把 `nvme0n1p1` 挂载到 `/data1`上，把 `nvme0n3p1` 挂载到 `/daat2` 上，把 `nvme0n4` 挂载到 `/data3` 上。

~~~bash
mkdir /data1
mkdir /data2
mkdir /data3

mount /dev/nvme0n1p1 /data1
mount /dev/nvme0n3p1 /data2
mount /dev/nvme0n4 /data3

# 查看 
[root@rocky /]# df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
devtmpfs       devtmpfs  4.0M     0  4.0M   0% /dev
tmpfs          tmpfs     979M     0  979M   0% /dev/shm
tmpfs          tmpfs     392M  6.7M  385M   2% /run
efivarfs       efivarfs  256K   33K  224K  13% /sys/firmware/efi/efivars
/dev/nvme0n2p3 xfs        18G  1.5G   16G   9% /
/dev/nvme0n2p1 vfat      500M  7.4M  493M   2% /boot/efi
tmpfs          tmpfs     196M     0  196M   0% /run/user/0
/dev/nvme0n1p1 xfs       960M   39M  922M   5% /data1
/dev/nvme0n3p1 ext4      974M   24K  907M   1% /data2
/dev/nvme0n4   xfs       5.0G   68M  4.9G   2% /data3
~~~



**注意**：1. 一个文件系统可以挂载到多个文件夹上，它们的数据源是一样的，在一个文件夹中修改数据，另一个文件夹的数据也会同步变化。2. 重新格式化制作文件系统，数据会全部消失。



#### 卸载文件系统

卸载时可以卸载挂载点，也可以卸载文件系统，效果是一样的。

~~~bash
umount /data1
umount /dev/nvme0n1p1

# 强制卸载
umount -l /data1
~~~

**注意**：一个文件系统如果已经存了数据，此时把它从一个挂载点卸载，然后重新挂载到一个新文件夹上。旧挂载点文件夹中数据没了，新文件夹内有将出现数据。即卸载文件系统，数据是不会丢失的，因为**数据不存在文件种，数据存在分区的硬盘上**。但是在这个分区上重新格式化制作文件系统那数据旧没了。





## 开机自动挂载

手动通过 `mount` 命令做的挂载都是一次性的，关机就失去了这些挂载点（数据没丢，相当于执行了 `umount`），想要永久挂载，可以使用开机自动挂载。开机自动挂载可以有两种方式：

1. 将挂载命令写到 `/etc/rc.local` 文件（这是一个软连接，需要把目标文件加上 `x` 执行权限）
2. 使用挂载任务专用的 `/etc/fstab` 配置文件【推荐】



### 使用 `/etc/rc.local` 文件

这个文件是一个软连接，指向 `/etc/rc.d/rc.local`

~~~bash
[root@rocky ~]# ls -l /etc/rc.local
lrwxrwxrwx. 1 root root 13 Apr 24 14:27 /etc/rc.local -> rc.d/rc.local
[root@rocky ~]# ls -l /etc/rc.d/rc.local
-rw-r--r--. 1 root root 474 Apr 24 14:27 /etc/rc.d/rc.local

# 增加挂载命令
echo "mount /dev/nvme0n1p1 /data1" >> /etc/rc.local

# 增加 x 执行权限

chmod +x /etc/rc.d/rc.local
~~~



### 使用 `/etc/fstab` 文件【推荐】

`/etc/fstab`（File System Table）是 Linux 系统中用于定义磁盘分区、存储设备或远程文件系统如何挂载的配置文件。系统启动时会自动读取此文件并按配置挂载文件系统。编辑 `/etc/fstab` 文件之后，执行 `systemctl daemon-reload` 就可以看到新挂载点了。

~~~bash
[root@rocky ~]# cat /etc/fstab

UUID=2873a616-3379-40ed-8b3e-8e253ade2ace /                       xfs     defaults        0 0
UUID=1517-C776          /boot/efi               vfat    umask=0077,shortname=winnt 0 2
UUID=980c85a9-be99-4bfa-9ab6-46052dc81a9e none                    swap    defaults        0 0

# 增加自己的挂载点
/dev/nvme0n3p1   /data2   ext4   defaults        0 0
~~~

>补充，`mount  -a` 挂载 /etc/fstab 中配置的所有挂载信息



###  文件 `/etc/fstab` 配置字段解释

在 `/etc/fatab` 文件中每行定义一个挂载项，共有 **6 个字段**，用空格或制表符分隔，格式如下：

```text
<设备标识> <挂载点> <文件系统类型> <挂载选项> <dump备份> <fsck检查顺序>
```

**1. 设备标识 (`<设备标识>`)**

指定要挂载的设备或存储，支持以下形式：
- **设备路径**：如 `/dev/sda1`（但可能因设备顺序变化导致问题）。
- **UUID**：唯一标识符（推荐），通过 `blkid` 或 `lsblk -f` 查看。 
  ```bash
  blkid /dev/sda1  # 查看设备的 UUID
  ```

**2. 挂载点 (`<挂载点>`)**
文件系统挂载到的目录路径（需提前创建目录）。

**3. 文件系统类型 (`<文件系统类型>`)**
指定文件系统格式，常见类型：

- `ext4`/`xfs`/`btrfs`：Linux 原生文件系统。
- `ntfs`/`vfat`：Windows 文件系统（需 `ntfs-3g` 或 `fat` 工具支持）。
- `nfs`/`cifs`：网络文件系统。
- `auto`：自动检测（不推荐，可能不安全）。

**4. 挂载选项 (`<挂载选项>`)**

控制挂载行为的逗号分隔选项，常用选项：

| **选项**   | **作用**                                                 |
| :--------- | :------------------------------------------------------- |
| `defaults` | 默认选项（包括 `rw,suid,dev,exec,auto,nouser,async`）。  |
| `rw`/`ro`  | 读写或只读挂载。                                         |
| `noexec`   | 禁止执行该分区中的程序（增强安全性）。                   |
| `nosuid`   | 忽略 SUID/SGID 权限（防提权）。                          |
| `user`     | 允许普通用户挂载（默认仅 root 可操作）。                 |
| `nofail`   | 启动时忽略挂载失败（避免因设备未就绪导致系统无法启动）。 |
| `discard`  | 启用 SSD TRIM 功能（优化固态硬盘性能）。                 |
| `_netdev`  | 网络设备（等网络就绪后再挂载，如 NFS/iSCSI）。           |

**5. dump 备份 (`<dump备份>`)**

是否被 `dump` 备份工具使用：`0`：禁用备份（默认）；`1`：允许备份（通常仅根目录设为 `1`）。

**6. fsck 检查顺序 (`<fsck检查顺序>`)**

系统启动时 `fsck` 检查文件系统的顺序：
- `0`：不检查（如非物理设备：`tmpfs`、`swap` 或网络存储）。
- `1`：优先检查（通常根目录设为 `1`）。
- `2+`：按数字顺序检查（其他分区设为 `2`）。





## 制作 swap 分区

Swap 分区（交换分区）是 Linux 系统在 **磁盘上预留的一块空间**，用于扩展系统的 **虚拟内存（Virtual Memory）**。当物理内存（RAM）不足时，系统会将部分不活跃的内存数据暂时存储到 Swap 分区，以释放 RAM 供更重要的进程使用。现代大内存机器可减少 Swap 依赖，但完全禁用可能增加系统不稳定风险。Swap 可以是一个 **独立的分区**（如 `/dev/sda2`），也可以是一个 **Swap 文件**（如 `/swapfile`）。

使用 `free` 查看内存情况，buff/cache 是操作系统从物理内存上借走的一部分用于操作批量写和缓存读的内存。avaliable展示的是真实可用的内存。使用 `sync` 命令通知操作系统抓紧把 buff 中的数据写到硬盘上，使用 `echo 3 > /proc/sys/vm/drop_caches` 清理 cache【生产环境慎用】。

~~~bash
[root@rocky ~]# free -h
               total        used        free      shared  buff/cache   available
Mem:           1.9Gi       263Mi       1.6Gi       5.0Mi       132Mi       1.7Gi
Swap:          2.0Gi          0B       2.0Gi
~~~

### 制作 swap 分区

找到一个未使用的分区，比如 `/dev/nvme0n1p5`，使用 `mkswap` 命令制作 swap 分区

~~~bash
# 制作 swap 分区
mkswap /dev/nvme0n1p5

# 激活 swap 分区
swapon /dev/nvme0n1p5

# 查看 swap 分区
[root@rocky ~]# swapon -s
Filename				Type		Size		Used		Priority
/dev/nvme0n2p2                          partition	2097148		0		-2
/dev/nvme0n1p5                          partition	2097148		0		-3


# 关闭某个 swap 分区
swapoff /dev/nvme0n1p5
~~~



### 使用文件制作 swap 分区

如果不能在插入新硬盘，也没有新的分区可以使用，那可以在现在文件系统资源充足的位置上新建一个文件夹，用文件夹做 swap 分区使用

~~~bash
# 准备文件
dd if=/dev/zero of=/swap_file bs=200M count=5

# 赋权
chmod 0600 /swap_file

# 使用 -f 制作文件 swap 分区
mkswap -f /swap_file

# 激活
swapon /swap_file
~~~



### 设置开机自动挂载 swap 分区

使用文件 `/etc/fstab` 

~~~bash
UUID=980c85a9-be99-4bfa-9ab6-46052dc81a9e      none               swap    defaults        0 0


# 对于文件 swap，第一列可以使用uuid 也可以使用文件绝对路径；第二列要使用文件绝对路径
# 文件swap也有 uuid，使用 mksap -f 时会自动输出
/swap_file   /swap_file                    swap    defaults        0 0
~~~

>查看分区的 uuid 使用 `blkid`





## xfs文件系统增量备份

待补充......



## LVM动态扩容

### lvm 是什么

### lvm 基本使用

### lvm 动态扩容

### lvm 快照

