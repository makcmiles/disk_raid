Сначала описание действий по методичке, с 14 пункта вывод скрипта автоматического запуска.

1. Подключенные диски после развёртывания ВМ
```
[vagrant@otuslinux ~]$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
sda      8:0    0   40G  0 disk 
`-sda1   8:1    0   40G  0 part /
sdb      8:16   0  250M  0 disk 
sdc      8:32   0  250M  0 disk 
sdd      8:48   0  250M  0 disk 
sde      8:64   0  250M  0 disk 
sdf      8:80   0  250M  0 disk
```
3. Создание RAID-массива 6-ого уровня
```
[vagrant@otuslinux ~]$ mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}
mdadm: must be super-user to perform this action
[vagrant@otuslinux ~]$ sudo mdadm --create --verbose /dev/md0 -l 6 -n 5 /dev/sd{b,c,d,e,f}
mdadm: layout defaults to left-symmetric
mdadm: layout defaults to left-symmetric
mdadm: chunk size defaults to 512K
mdadm: size set to 253952K
mdadm: Defaulting to version 1.2 metadata
mdadm: array /dev/md0 started.
```
5. Как это выглядит в системе
```
sblk
NAME   MAJ:MIN RM  SIZE RO TYPE  MOUNTPOINT
sda      8:0    0   40G  0 disk  
`-sda1   8:1    0   40G  0 part  /
sdb      8:16   0  250M  0 disk  
`-md0    9:0    0  744M  0 raid6 
sdc      8:32   0  250M  0 disk  
`-md0    9:0    0  744M  0 raid6 
sdd      8:48   0  250M  0 disk  
`-md0    9:0    0  744M  0 raid6 
sde      8:64   0  250M  0 disk  
`-md0    9:0    0  744M  0 raid6 
sdf      8:80   0  250M  0 disk  
`-md0    9:0    0  744M  0 raid6 
```
```
[vagrant@otuslinux ~]$ cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid6 sdf[4] sde[3] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]
```
```
vagrant@otuslinux ~]$ sudo mdadm --detail /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Nov 23 18:32:03 2023
        Raid Level : raid6
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu Nov 23 18:32:10 2023
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : f430a63a:d92a745e:7e624ed9:f1a4c817
            Events : 17

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       3       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
```
4. Создание mdadm.conf 
```
[vagrant@otuslinux mdadm]$ sudo -i
[root@otuslinux ~]# echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
[root@otuslinux mdadm]# mdadm --detail --scan --verbose | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf
```
5. Ломаем RAID
```
[root@otuslinux mdadm]# mdadm /dev/md0 --fail /dev/sde
mdadm: set /dev/sde faulty in /dev/md0
```
```
[root@otuslinux mdadm]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid6 sdf[4] sde[3](F) sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/4] [UUU_U]
```
```
mdadm -D /dev/md0
Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       -       0        0        3      removed
       4       8       80        4      active sync   /dev/sdf

       3       8       64        -      faulty   /dev/sde
```
6. Удаляем устройство
```
[root@otuslinux mdadm]# mdadm /dev/md0 --remove /dev/sde
mdadm: hot removed /dev/sde from /dev/md0
```
```
mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Nov 23 18:32:03 2023
        Raid Level : raid6
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 4
       Persistence : Superblock is persistent

       Update Time : Thu Nov 23 19:46:58 2023
             State : clean, degraded 
    Active Devices : 4
   Working Devices : 4
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : f430a63a:d92a745e:7e624ed9:f1a4c817
            Events : 20

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       -       0        0        3      removed
       4       8       80        4      active sync   /dev/sdf
```
7. Добавляем устройство
```
[root@otuslinux mdadm]# mdadm /dev/md0 --add /dev/sde
mdadm: added /dev/sde
```
8. Проверяем
```
[root@otuslinux mdadm]# cat /proc/mdstat
Personalities : [raid6] [raid5] [raid4] 
md0 : active raid6 sde[5] sdf[4] sdd[2] sdc[1] sdb[0]
      761856 blocks super 1.2 level 6, 512k chunk, algorithm 2 [5/5] [UUUUU]
      
unused devices: <none>
```
```
[root@otuslinux mdadm]# mdadm -D /dev/md0
/dev/md0:
           Version : 1.2
     Creation Time : Thu Nov 23 18:32:03 2023
        Raid Level : raid6
        Array Size : 761856 (744.00 MiB 780.14 MB)
     Used Dev Size : 253952 (248.00 MiB 260.05 MB)
      Raid Devices : 5
     Total Devices : 5
       Persistence : Superblock is persistent

       Update Time : Thu Nov 23 19:50:25 2023
             State : clean 
    Active Devices : 5
   Working Devices : 5
    Failed Devices : 0
     Spare Devices : 0

            Layout : left-symmetric
        Chunk Size : 512K

Consistency Policy : resync

              Name : otuslinux:0  (local to host otuslinux)
              UUID : f430a63a:d92a745e:7e624ed9:f1a4c817
            Events : 39

    Number   Major   Minor   RaidDevice State
       0       8       16        0      active sync   /dev/sdb
       1       8       32        1      active sync   /dev/sdc
       2       8       48        2      active sync   /dev/sdd
       5       8       64        3      active sync   /dev/sde
       4       8       80        4      active sync   /dev/sdf
```
9. Создаю таблицу GPT
```
parted -s /dev/md0 mklabel GPT

[root@otuslinux mdadm]# parted -l 
odel: Linux Software RAID Array (md)
Disk /dev/md0: 780MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:
```
10. Создание разделов и файловых систем
```
[root@otuslinux mdadm]# parted /dev/md0 mkpart primary ext4 0% 20%
Information: You may need to update /etc/fstab.

[root@otuslinux mdadm]# parted /dev/md0 mkpart primary ext4 20% 40%
Information: You may need to update /etc/fstab.

[root@otuslinux mdadm]# parted /dev/md0 mkpart primary ext4 40% 60%       
Information: You may need to update /etc/fstab.

[root@otuslinux mdadm]# parted /dev/md0 mkpart primary ext4 60% 80%       
Information: You may need to update /etc/fstab.

[root@otuslinux mdadm]# parted /dev/md0 mkpart primary ext4 80% 100%      
Information: You may need to update /etc/fstab.
```
```
[root@otuslinux mdadm]# parted -l

Model: Linux Software RAID Array (md)
Disk /dev/md0: 780MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags: 

Number  Start   End    Size   File system  Name     Flags
 1      1573kB  156MB  154MB               primary
 2      156MB   311MB  156MB               primary
 3      311MB   469MB  157MB               primary
 4      469MB   624MB  156MB               primary
 5      624MB   779MB  154MB               primary
```
```
[root@otuslinux mdadm]# for i in $(seq 1 5); do sudo mkfs.ext4 /dev/md0p$i; done
mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
37696 inodes, 150528 blocks
7526 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
1984 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38152 inodes, 152064 blocks
7603 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38456 inodes, 153600 blocks
7680 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2024 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
38152 inodes, 152064 blocks
7603 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
2008 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done 

mke2fs 1.42.9 (28-Dec-2013)
Filesystem label=
OS type: Linux
Block size=1024 (log=0)
Fragment size=1024 (log=0)
Stride=512 blocks, Stripe width=1536 blocks
37696 inodes, 150528 blocks
7526 blocks (5.00%) reserved for the super user
First data block=1
Maximum filesystem blocks=33816576
19 block groups
8192 blocks per group, 8192 fragments per group
1984 inodes per group
Superblock backups stored on blocks: 
	8193, 24577, 40961, 57345, 73729

Allocating group tables: done                            
Writing inode tables: done                            
Creating journal (4096 blocks): done
Writing superblocks and filesystem accounting information: done
```
12. Монирую разделы
```
[root@otuslinux mdadm]# mkdir -p /raid/part{1,2,3,4,5}
[root@otuslinux mdadm]# for i in $(seq 1 5); do mount /dev/md0p$i /raid/part$i; done
```
13. Вывод
```
[root@otuslinux mdadm]# df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        489M     0  489M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M  6.7M  489M   2% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
/dev/sda1        40G  4.4G   36G  11% /
tmpfs           100M     0  100M   0% /run/user/1000
/dev/md0p1      139M  1.6M  127M   2% /raid/part1
/dev/md0p2      140M  1.6M  128M   2% /raid/part2
/dev/md0p3      142M  1.6M  130M   2% /raid/part3
/dev/md0p4      140M  1.6M  128M   2% /raid/part4
/dev/md0p5      139M  1.6M  127M   2% /raid/part5
```
```
[root@otuslinux mdadm]# lsblk 
NAME      MAJ:MIN RM   SIZE RO TYPE  MOUNTPOINT
sda         8:0    0    40G  0 disk  
`-sda1      8:1    0    40G  0 part  /
sdb         8:16   0   250M  0 disk  
`-md0       9:0    0   744M  0 raid6 
  |-md0p1 259:0    0   147M  0 md    /raid/part1
  |-md0p2 259:5    0 148.5M  0 md    /raid/part2
  |-md0p3 259:6    0   150M  0 md    /raid/part3
  |-md0p4 259:7    0 148.5M  0 md    /raid/part4
  `-md0p5 259:8    0   147M  0 md    /raid/part5
sdc         8:32   0   250M  0 disk  
`-md0       9:0    0   744M  0 raid6 
  |-md0p1 259:0    0   147M  0 md    /raid/part1
  |-md0p2 259:5    0 148.5M  0 md    /raid/part2
  |-md0p3 259:6    0   150M  0 md    /raid/part3
  |-md0p4 259:7    0 148.5M  0 md    /raid/part4
  `-md0p5 259:8    0   147M  0 md    /raid/part5
sdd         8:48   0   250M  0 disk  
`-md0       9:0    0   744M  0 raid6 
  |-md0p1 259:0    0   147M  0 md    /raid/part1
  |-md0p2 259:5    0 148.5M  0 md    /raid/part2
  |-md0p3 259:6    0   150M  0 md    /raid/part3
  |-md0p4 259:7    0 148.5M  0 md    /raid/part4
  `-md0p5 259:8    0   147M  0 md    /raid/part5
sde         8:64   0   250M  0 disk  
`-md0       9:0    0   744M  0 raid6 
  |-md0p1 259:0    0   147M  0 md    /raid/part1
  |-md0p2 259:5    0 148.5M  0 md    /raid/part2
  |-md0p3 259:6    0   150M  0 md    /raid/part3
  |-md0p4 259:7    0 148.5M  0 md    /raid/part4
  `-md0p5 259:8    0   147M  0 md    /raid/part5
sdf         8:80   0   250M  0 disk  
`-md0       9:0    0   744M  0 raid6 
  |-md0p1 259:0    0   147M  0 md    /raid/part1
  |-md0p2 259:5    0 148.5M  0 md    /raid/part2
  |-md0p3 259:6    0   150M  0 md    /raid/part3
  |-md0p4 259:7    0 148.5M  0 md    /raid/part4
  `-md0p5 259:8    0   147M  0 md    /raid/part5
```

14. При автоматизации создания прописал ссылку на скрпит в Вагрант-файл, выбрал создание RAID-5. Вывод после создания ВМ
```
[vagrant@otuslinux ~]$ df -h
Filesystem      Size  Used Avail Use% Mounted on
devtmpfs        489M     0  489M   0% /dev
tmpfs           496M     0  496M   0% /dev/shm
tmpfs           496M  6.7M  489M   2% /run
tmpfs           496M     0  496M   0% /sys/fs/cgroup
/dev/sda1        40G  4.4G   36G  11% /
/dev/md0p1      186M  1.6M  171M   1% /raid/part1
/dev/md0p2      188M  1.6M  173M   1% /raid/part2
/dev/md0p3      190M  1.6M  175M   1% /raid/part3
/dev/md0p4      188M  1.6M  173M   1% /raid/part4
/dev/md0p5      186M  1.6M  171M   1% /raid/part5
tmpfs           100M     0  100M   0% /run/user/1000
``` 
