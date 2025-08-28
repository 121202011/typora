### Linux 系统 LVM 磁盘管理完整操作指南

本文基于 `node110` 服务器环境，详细记录从磁盘分区、LVM 组件创建（PV→VG→LV）、挂载使用，到后续扩容与删除的全流程，包含每步实操命令、输出结果及关键说明，适合需要灵活管理磁盘空间的场景参考。

#### 一、环境初始状态：查看磁盘信息

首先通过 `lsblk` 命令确认服务器当前磁盘布局，明确待操作的目标磁盘（此处为 `/dev/sdb`，容量 20G）：

```bash
[root@node110 ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  200G  0 disk 
├─sda1   8:1    0  300M  0 part /boot
├─sda2   8:2    0    2G  0 part [SWAP]
└─sda3   8:3    0 197.7G 0 part /
sdb      8:16   0   20G  0 disk 
└─sdb1   8:17   0   20G  0 part  # 初始为单分区，需重新划分
sdc      8:32   0   20G  0 disk 
sr0     11:0    1 1024M  0 rom  
```

#### 二、步骤 1：磁盘分区（以 `/dev/sdb` 为例）

需删除 `/dev/sdb` 原有分区，重新创建 **1 个 1G 主分区（sdb1）+1 个 2G 主分区（sdb2）**，操作如下：

##### 1. 进入分区工具

```bash
[root@node110 ~]# fdisk /dev/sdb
Welcome to fdisk (util-linux 2.23.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.
```

##### 2. 删除原有分区（sdb1）

```bash
Command (m for help): p  # 查看当前分区
Disk /dev/sdb: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disk label type: dos
Disk identifier: 0x211508c7
Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048    41943039    20970496   83  Linux

Command (m for help): d  # 删除分区
Selected partition 1
Partition 1 is deleted
```

##### 3. 创建新分区（sdb1：1G，sdb2：2G）

```bash
# 创建第1个主分区（1G）
Command (m for help): n
Partition type:
   p   primary (0 primary, 0 extended, 4 free)
   e   extended
Select (default p): p  # 选择主分区
Partition number (1-4, default 1): 1  # 分区号1
First sector (2048-41943039, default 2048):  # 起始扇区默认（回车）
Using default value 2048
Last sector, +sectors or +size{K,M,G} (2048-41943039, default 41943039): +1G  # 大小1G
Partition 1 of type Linux and of size 1 GiB is set

# 创建第2个主分区（2G）
Command (m for help): n
Partition type:
   p   primary (1 primary, 0 extended, 3 free)
   e   extended
Select (default p): p
Partition number (2-4, default 2): 2  # 分区号2
First sector (2099200-41943039, default 2099200):  # 起始扇区默认（回车）
Using default value 2099200
Last sector, +sectors or +size{K,M,G} (2099200-41943039, default 41943039): +2G  # 大小2G
Partition 2 of type Linux and of size 2 GiB is set

# 保存分区设置
Command (m for help): w
The partition table has been altered!
Calling ioctl() to re-read partition table.
Syncing disks.
```

##### 4. 验证分区结果

```bash
[root@node110 ~]# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0  200G  0 disk 
├─sda1   8:1    0  300M  0 part /boot
├─sda2   8:2    0    2G  0 part [SWAP]
└─sda3   8:3    0 197.7G 0 part /
sdb      8:16   0   20G  0 disk 
├─sdb1   8:17   0    1G  0 part  # 新分区1（1G）
└─sdb2   8:18   0    2G  0 part  # 新分区2（2G）
sdc      8:32   0   20G  0 disk 
sr0     11:0    1 1024M  0 rom  
```

#### 三、步骤 2：创建 LVM 组件（PV→VG→LV）

LVM（逻辑卷管理）需按「物理卷（PV）→卷组（VG）→逻辑卷（LV）」的顺序创建，将分区转化为可灵活管理的逻辑空间。

##### 1. 创建物理卷（PV）

将 `/dev/sdb1` 和 `/dev/sdb2` 初始化为 LVM 物理卷：

```bash
# 创建PV
[root@node110 ~]# pvcreate /dev/sdb1 /dev/sdb2
Physical volume "/dev/sdb1" successfully created.
Physical volume "/dev/sdb2" successfully created.

# 查看PV信息（简洁版：pvs）
[root@node110 ~]# pvs
  PV         VG   Fmt  Attr PSize  PFree 
  /dev/sdb1       lvm2 ---  1.00g  1.00g
  /dev/sdb2       lvm2 ---  2.00g  2.00g

# 查看PV信息（详细版：pvdisplay）
[root@node110 ~]# pvdisplay
  "/dev/sdb1" is a new physical volume of "1.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb1
  VG Name               
  PV Size               1.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               n8uAAH-vusA-aFnV-fwac-KXJY-XNXX-WgNQLC

  "/dev/sdb2" is a new physical volume of "2.00 GiB"
  --- NEW Physical volume ---
  PV Name               /dev/sdb2
  VG Name               
  PV Size               2.00 GiB
  Allocatable           NO
  PE Size               0   
  Total PE              0
  Free PE               0
  Allocated PE          0
  PV UUID               Lfgnt0-4deT-sr6e-Ju0K-mSaK-tsIy-vVHA9n
```

##### 2. 创建卷组（VG）

将 2 个 PV 加入新卷组 `vgdata`（卷组名可自定义）：

```bash
# 创建VG
[root@node110 ~]# vgcreate vgdata /dev/sdb1 /dev/sdb2
Volume group "vgdata" successfully created

# 查看VG信息（简洁版：vgs）
[root@node110 ~]# vgs
  VG     #PV #LV #SN Attr   VSize VFree
  vgdata   2   0   0 wz--n- 2.99g 2.99g  # 总容量2.99G，空闲2.99G

# 查看VG信息（详细版：vgdisplay）
[root@node110 ~]# vgdisplay
  --- Volume group ---
  VG Name               vgdata
  System ID             
  Format                lvm2
  Metadata Areas        2
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                2
  Act PV                2
  VG Size               2.99 GiB
  PE Size               4.00 MiB
  Total PE              766
  Alloc PE / Size       0 / 0   
  Free  PE / Size       766 / 2.99 GiB
  VG UUID               uhbe0H-y3PG-375s-p6sI-i0bh-DdCG-Per1k8
```

##### 3. 创建逻辑卷（LV）

从 `vgdata` 卷组中划分 **500M 空间**，创建逻辑卷 `lvdata`（LV 名可自定义）：

```bash
# 创建LV
[root@node110 ~]# lvcreate -L 500M -n lvdata vgdata
Logical volume "lvdata" created.

# 查看LV信息（简洁版：lvs）
[root@node110 ~]# lvs
  LV     VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  lvdata vgdata -wi-a----- 500.00m                                                    
  
# 查看LV信息（详细版：lvdisplay）
[root@node110 ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/vgdata/lvdata  # LV设备路径（关键，后续需用到）
  LV Name                lvdata
  VG Name                vgdata
  LV UUID                7MwJuX-xHHh-Xnjh-quz4-Hd0j-LUG3-aIn3iy
  LV Write Access        read/write
  LV Creation host, time node110, 2022-03-09 12:23:25-0800
  LV Status              available
  # open                 0
  LV Size                500.00 MiB
  Current LE             125
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     8192
  Block device           253:0
```

#### 四、步骤 3：使用逻辑卷（格式化 + 挂载）

创建好的 LV 需格式化（赋予文件系统）并挂载到目录，才能用于存储数据。

##### 1. 格式化 LV（ext4 文件系统）

此处选择 `ext4` 格式（兼容广、稳定性高），也可根据需求选择 `xfs`（需用 `mkfs.xfs` 命令）：

```bash
[root@node110 ~]# mkfs.ext4 /dev/vgdata/lvdata
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=0 blocks, Stripe width=0 blocks
128016 inodes, 512000 blocks
25600 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=34078720
63 block groups
8192 blocks per group, 8192 fragments per group
2032 inodes per group
Superblock backups stored on blocks: 
  8193, 24577, 40961, 57345, 73729, 204801, 221185, 401409
Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (8192 blocks): done
Writing superblocks and filesystem accounting information: done  
```

##### 2. 创建挂载点并手动挂载

选择 `/images` 作为挂载点（目录名可自定义），将 LV 挂载到该目录：

```bash
# 创建挂载点
[root@node110 ~]# mkdir /images

# 挂载LV到/images
[root@node110 ~]# mount /dev/vgdata/lvdata /images/

# 验证挂载结果（df -h 查看已挂载的文件系统）
[root@node110 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
/dev/sda3                     198G   17G  182G   9% /
devtmpfs                      471M     0  471M   0% /dev
tmpfs                         487M     0  487M   0% /dev/shm
tmpfs                         487M  8.1M  479M   2% /run
tmpfs                         487M     0  487M   0% /sys/fs/cgroup
/dev/sda1                     297M  147M  151M  50% /boot
tmpfs                          98M     0   98M   0% /run/user/0
/dev/mapper/vgdata-lvdata     477M  2.3M  445M   1% /images  # 已成功挂载
```

##### 3. 配置开机自动挂载（/etc/fstab）

手动挂载仅临时生效，需编辑 `/etc/fstab` 文件，确保服务器重启后自动挂载：

```bash
# 编辑/etc/fstab
[root@node110 ~]# vim /etc/fstab

# 在文件末尾添加以下内容（格式：设备路径 挂载点 文件系统 选项 0 0）
/dev/vgdata/lvdata  /images  ext4  defaults  0  0
```

##### 4. 验证自动挂载配置

通过 `umount` 卸载后，再用 `mount -a` 重新加载 `/etc/fstab`，确认挂载正常：

```bash
# 卸载LV
[root@node110 ~]# umount /images

# 重新加载/etc/fstab（自动挂载）
[root@node110 ~]# mount -a

# 再次验证
[root@node110 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
...
/dev/mapper/vgdata-lvdata     477M  2.3M  445M   1% /images  # 自动挂载成功
```

#### 五、步骤 4：逻辑卷扩容（两种场景）

当 `/images` 空间不足时，需对 LV 进行扩容，分「卷组有空闲空间」和「卷组无空闲空间」两种场景处理。

##### 场景 1：卷组（vgdata）有足够空闲空间

若 `vgdata` 仍有未分配空间（如本例初始空闲 2.99G，创建 LV 后剩余～2.5G），直接扩展 LV 即可：

```bash
# 1. 扩展LV：增加500M空间（-L +500M 表示在原有基础上加500M）
[root@node110 ~]# lvextend -L +500M /dev/vgdata/lvdata
  Size of logical volume vgdata/lvdata changed from 500.00 MiB (125 extents) to 1000.00 MiB (250 extents).
  Logical volume vgdata/lvdata successfully resized.

# 2. 同步文件系统（ext4 用 resize2fs，xfs 用 xfs_growfs）
[root@node110 ~]# resize2fs /dev/mapper/vgdata-lvdata
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/mapper/vgdata-lvdata is mounted on /images; on-line resizing required
old_desc_blocks = 4, new_desc_blocks = 8
The filesystem on /dev/mapper/vgdata-lvdata is now 1024000 blocks long.

# 3. 验证扩容结果（LV 已从 500M 扩展到 ~1G）
[root@node110 ~]# df -h
Filesystem                    Size  Used Avail Use% Mounted on
/dev/mapper/vgdata-lvdata     961M  2.5M  910M   1% /images
```

##### 场景 2：卷组（vgdata）无空闲空间

若 `vgdata` 空间已耗尽，需先新增分区（如 `/dev/sdb3`），再将其加入卷组，最后扩展 LV：

###### 1. 新增分区（sdb3：2G）

```bash
[root@node110 ~]# fdisk /dev/sdb
Command (m for help): p  # 查看当前分区
Disk /dev/sdb: 21.5 GB, 21474836480 bytes, 41943040 sectors
Units = sectors of 1 * 512 = 512 bytes
Device Boot      Start         End      Blocks   Id  System
/dev/sdb1            2048     2099199     1048576   83  Linux
/dev/sdb2         2099200     6293503     2097152   83  Linux

# 创建第3个主分区（2G）
Command (m for help): n
Partition type:
   p   primary (2 primary, 0 extended, 2 free)
   e   extended
Select (default p): p
Partition number (3-4, default 3): 3
First sector (6293504-41943039, default 6293504):  # 默认回车
Using default value 6293504
Last sector, +sectors or +size{K,M,G} (6293504-41943039, default 41943039): +2G  # 2G
Partition 3 of type Linux and of size 2 GiB is set

# 保存分区
Command (m for help): w
The partition table has been altered!
Calling ioctl() to re-read partition table.
WARNING: Re-reading the partition table failed with error 16: Device or resource busy.
The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or kpartx(8)
Syncing disks.
```

###### 2. 刷新分区表（无需重启）

```bash
[root@node110 ~]# partprobe  # 刷新分区表，让系统识别新分区sdb3

# 验证新分区
[root@node110 ~]# lsblk
sdb      8:16   0   20G  0 disk 
├─sdb1   8:17   0    1G  0 part 
├─sdb2   8:18   0    2G  0 part 
└─sdb3   8:19   0    2G  0 part  # 新分区sdb3（2G）
```

###### 3. 将新分区加入卷组（vgdata）

```bash
[root@node110 ~]# vgextend vgdata /dev/sdb3  # 扩展卷组
  Physical volume "/dev/sdb3" successfully created.
  Volume group "vgdata" successfully extended

# 验证卷组空间（空闲空间已增加2G，总空闲~4G）
[root@node110 ~]# vgs
  VG     #PV #LV #SN Attr   VSize VFree
  vgdata   3   1   0 wz--n- 4.99g 4.01g
```

###### 4. 扩展 LV 至 2G（或按需调整大小）

```bash
# 扩展LV到2G（-L 2G 表示直接将LV大小设为2G）
[root@node110 ~]# lvextend -L 2G /dev/vgdata/lvdata
  Size of logical volume vgdata/lvdata changed from 1000.00 MiB (250 extents) to 2.00 GiB (512 extents).
  Logical volume vgdata/lvdata successfully resized.

# 同步文件系统
[root@node110 ~]# resize2fs /dev/vgdata/lvdata
resize2fs 1.42.9 (28-Dec-2013)
Filesystem at /dev/vgdata/lvdata is mounted on /images; on-line resizing required
old_desc_blocks = 8, new_desc_blocks = 16
The filesystem on /dev/vgdata/lvdata is now 2097152 blocks long.

# 验证结果（LV 已扩展到 ~2G）
[root@node110 ~]# df -h
/dev/mapper/vgdata-lvdata     2.0G  3.0M  1.9G   1% /images
```

#### 六、步骤 5：LVM 完整删除流程（谨慎操作）

若需清理 LVM 组件（如更换磁盘），需按「备份数据→删除自动挂载→卸载→删除 LV→删除 VG→删除 PV→删除分区」的顺序操作，避免数据丢失或系统异常。

##### 1. 备份数据（关键！）

若 `/images` 目录有重要数据，需先备份到其他位置（如 `/backup`）：

```bash
[root@node110 ~]# cp -r /images/* /backup/
```

##### 2. 删除 `/etc/fstab` 中的自动挂载配置

```bash
[root@node110 ~]# vim /etc/fstab
# 删除以下行：
# /dev/vgdata/lvdata  /images  ext4  defaults  0  0
```

##### 3. 卸载逻辑卷

```bash
[root@node110 ~]# umount /images
```

##### 4. 删除逻辑卷（LV）

```bash
[root@node110 ~]# lvremove /dev/vgdata/lvdata
Do you really want to remove active logical volume vgdata/lvdata? [y/n]: y  # 确认删除
  Logical volume "lvdata" successfully removed
```

##### 5. 禁用并删除卷组（VG）

```bash
# 禁用卷组
[root@node110 ~]# vgchange -an vgdata
  0 logical volume(s) in volume group "vgdata" now active

# 删除卷组
[root@node110 ~]# vgremove vgdata
  Volume group "vgdata" successfully removed
```

##### 6. 删除物理卷（PV）

```bash
[root@node110 ~]# pvremove /dev/sdb1 /dev/sdb2 /dev/sdb3
  Labels on physical volume "/dev/sdb1" successfully wiped.
  Labels on physical volume "/dev/sdb2" successfully wiped.
  Labels on physical volume "/dev/sdb3" successfully wiped.
```

##### 7. 删除分区（可选，若需复用磁盘）

```bash
[root@node110 ~]# fdisk /dev/sdb
# 依次删除 sdb1、sdb2、sdb3（用 d 命令），最后 w 保存
Command (m for help): d
Partition number (1-3, default 3): 3
Partition 3 is deleted

Command (m for help): d
Partition number (1-2, default 2): 2
Partition 2 is deleted

Command (m for help): d
Partition number (1, default 1): 1
Partition 1 is deleted

Command (m for help): w
The partition table has been altered!
Syncing disks.

# 刷新分区表
[root@node110 ~]# partprobe
```

#### 七、扩展：磁盘 / 分区故障数据迁移

若卷组中某磁盘 / 分区（如 `/dev/sdb1`）故障，可通过以下步骤快速迁移数据，避免业务中断：

1. **迁移故障分区数据**：将 `/dev/sdb1` 的数据转移到同卷组其他健康 PV（如 `/dev/sdb2`）：

   ```bash
   [root@node110 ~]# pvmove /dev/sdb1 /dev/sdb2
   /dev/sdb1: Moved: 100.0%  # 迁移完成
   ```

2. **从卷组移除故障分区**：

   ```bash
   [root@node110 ~]# vgreduce vgdata /dev/sdb1
   Removed "/dev/sdb1" from volume group "vgdata"
   ```

3. **删除故障分区的 PV 标识**：

   ```bash
   [root@node110 ~]# pvremove /dev/sdb1
   Labels on physical volume "/dev/sdb1" successfully wiped.
   ```

4. **物理处理故障磁盘**：关机后更换故障磁盘，或通过工具修复分区。

#### 八、常见问题总结

1. **格式化时提示 “已有文件系统”**：
   若 LV 曾用 `ext4` 格式，现需改为 `xfs`，需加 `-f` 强制覆盖：

   ```bash
   [root@node110 ~]# mkfs.xfs -f /dev/vgdata/lvdata
   ```

2. **扩容后 `df -h` 不显示新空间**：
   忘记同步文件系统！`ext4` 用 `resize2fs`，`xfs` 用 `xfs_growfs 挂载点`（如 `xfs_growfs /images`）。

3. **分区后系统不识别新分区**：
   执行 `partprobe` 刷新分区表，无需重启服务器。