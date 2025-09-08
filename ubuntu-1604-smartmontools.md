# Testing disks with Ubuntu 16.04 and smartmontools

SATA SSD vs SATA RAID10 speed test with ~10 year old (date now 19.3.2021) enterprise server hardware

 List of S.M.A.R.T. attributes and definition

* <https://en.wikipedia.org/wiki/S.M.A.R.T.#Known_ATA_S.M.A.R.T._attributes>

Instructions for testing with FreeNAS

* <https://www.ixsystems.com/community/threads/hard-drive-burn-in-testing-discussion-thread.21451/>

## Prepare Ubuntu 16.04 when rebooted with Live USB

~~~sh
# Set password for user ubuntu
passwd
sudo apt-get update
# Fix apt-get database/cache error
sudo touch /var/cache/app-info/xapian/default
sudo apt-get update; \
sudo apt-get install -y openssh-server tmux smartmontools sg3-utils
sudo apt-get -y upgrade
~~~

## Commands for listing devices

~~~sh
ls -la /dev/sg*
sudo fdisk -l
df -h
# Check sg* <-> sd* mapping goes
sudo sg_map

# Examples
ubuntu@ubuntu:~$ ls -la /dev/sg*
crw-rw----  1 root disk  21,  0 Feb 11 16:10 /dev/sg0
crw-rw----  1 root disk  21,  1 Feb 11 16:10 /dev/sg1
crw-rw----  1 root disk  21, 10 Feb 11 16:10 /dev/sg10
crw-rw----  1 root disk  21, 11 Feb 11 16:10 /dev/sg11
crw-rw----  1 root disk  21, 12 Feb 11 16:10 /dev/sg12
crw-------  1 root root  21, 13 Feb 11 16:10 /dev/sg13
crw-rw----  1 root disk  21, 14 Feb 11 16:10 /dev/sg14
crw-rw----  1 root disk  21,  2 Feb 11 16:10 /dev/sg2
crw-rw----+ 1 root cdrom 21,  3 Feb 11 16:10 /dev/sg3
crw-rw----  1 root disk  21,  4 Feb 11 16:10 /dev/sg4
crw-rw----  1 root disk  21,  5 Feb 11 16:10 /dev/sg5
crw-rw----  1 root disk  21,  6 Feb 11 16:10 /dev/sg6
crw-rw----  1 root disk  21,  7 Feb 11 16:10 /dev/sg7
crw-rw----  1 root disk  21,  8 Feb 11 16:10 /dev/sg8
crw-rw----  1 root disk  21,  9 Feb 11 16:10 /dev/sg9

ubuntu@ubuntu:~$ sudo fdisk -l
Disk /dev/loop0: 1.5 GiB, 1601015808 bytes, 3126984 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/sda: 139.6 GiB, 149831024640 bytes, 292638720 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/sdb: 3.7 TiB, 3998614552576 bytes, 7809794048 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes

Disk /dev/sdc: 14.4 GiB, 15481176064 bytes, 30236672 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x3a4cd293

Device     Boot Start      End  Sectors  Size Id Type
/dev/sdc1  *     2048 30236671 30234624 14.4G  c W95 FAT32 (LBA)

ubuntu@ubuntu:~$ df -h
Filesystem      Size  Used Avail Use% Mounted on
udev             24G     0   24G   0% /dev
tmpfs           4.8G  9.6M  4.8G   1% /run
/dev/sdc1        15G  1.6G   13G  11% /cdrom
/dev/loop0      1.5G  1.5G     0 100% /rofs
aufs             24G  3.3G   21G  14% /
tmpfs            24G  340K   24G   1% /dev/shm
tmpfs           5.0M  4.0K  5.0M   1% /run/lock
tmpfs            24G     0   24G   0% /sys/fs/cgroup
tmpfs            24G   32K   24G   1% /tmp
tmpfs           4.8G   92K  4.8G   1% /run/user/999

# Output when disks in Morphed JBOD mode
# Adaptec RAID controller, create first JBOD and then manage -> convert to Morhped JBOD Mode with ctrl + v 
# badblocks test can be run only with this disk mode if one wants to test all disks at parallel.
# In RAID10 there will be error "badblocks: invalid starting block (0): must be less than 0" if trying to test /dev/sg*
ubuntu@ubuntu:~$ sudo sg_map
/dev/sg0  /dev/sda
/dev/sg1
/dev/sg2
/dev/sg3  /dev/sr0
/dev/sg4  /dev/sdb
/dev/sg5  /dev/sdc
/dev/sg6  /dev/sdd
/dev/sg7  /dev/sde
/dev/sg8  /dev/sdf
/dev/sg9  /dev/sdg
/dev/sg10  /dev/sdh
/dev/sg11  /dev/sdi
/dev/sg12
/dev/sg13
/dev/sg14
/dev/sg15
/dev/sg16
/dev/sg17
/dev/sg18
/dev/sg19
/dev/sg20
/dev/sg21  /dev/sdj

# Output when sda is a RAID 10 volume containing disks sg1 and sg2
# and sdb is a RAID 10 volume containig disks sg5-sg12.
# This is output of Ubuntu 16.04. Ubuntu 18.04 and 20.04 didn't show individual sg* disks only sg4 = sdb. This is probably due to super old hardware.
# smartmontools disk tests can be run with Ubuntu 16.04 and RAID10.
ubuntu@ubuntu:~$ sudo sg_map
/dev/sg0  /dev/sda
/dev/sg1
/dev/sg2
/dev/sg3  /dev/sr0
/dev/sg4  /dev/sdb
/dev/sg5
/dev/sg6
/dev/sg7
/dev/sg8
/dev/sg9
/dev/sg10
/dev/sg11
/dev/sg12
/dev/sg13
/dev/sg14  /dev/sdc
~~~

## Disk tests

* S.M.A.R.T

  ~~~sh
  # Check the disk hardware
  sudo smartctl --scan
  sudo smartctl -t short /dev/sg5
  sudo smartctl -t conveyance /dev/sg5
  sudo smartctl -t long /dev/sg5

  # One command if the disks are sg5-sg12 like in the output of sg_map command above.
  # This doesn't work if diks is in JBOD mode, they need to be in RAID10.
  for i in {5..12}; do sudo smartctl -t conveyance -d sat /dev/sg"$i"; done

  # If using SAS disks, use option -d scsi
  ~~~

* With SATA disk, you can check if self-test is over with command:

  ~~~sh
  sudo smartctl -a -d sat /dev/sg11 | grep -A1 -m1 Self-test

  Self-test execution status:      ( 249) Self-test routine in progress...
                                          90% of test remaining.

  # Check all in one go:
  for i in {5..12}; do sudo smartctl -a -d sat /dev/sg"$i" | grep -A1 -m1 Self-test; done
  ~~~

* Test badblocks
  * The disks need to be in JBOD mode. RAID10 will not work.
  * After badblocks run smartctl tests again.
  * Use tmux for creating many sessions. You can split screen with `ctrl+b then "`
  * This will erase any data on the disk unrecovable!

  ~~~sh
  tmux
  disk_sd="sd5"; sudo badblocks -ws /dev/$disk_sd > /tmp/badblocks_$disk_sd
  # To detach:
  # ctrl+b then d
  # To attach back:
  tmux attach
  ~~~

## Analyzing output information

* Example of SMART attribute output command:
  * smartctl -a -d sat /dev/sg5

  ~~~sh
  SMART Attributes Data Structure revision number: 16
  Vendor Specific SMART Attributes with Thresholds:
  ID# ATTRIBUTE_NAME          FLAG     VALUE WORST THRESH TYPE      UPDATED  WHEN_FAILED RAW_VALUE
    1 Raw_Read_Error_Rate     0x000f   200   200   051    Pre-fail  Always       -       0
    3 Spin_Up_Time            0x0003   199   198   021    Pre-fail  Always       -       3033
    4 Start_Stop_Count        0x0032   100   100   000    Old_age   Always       -       52
    5 Reallocated_Sector_Ct   0x0033   200   200   140    Pre-fail  Always       -       0
    7 Seek_Error_Rate         0x000e   200   200   000    Old_age   Always       -       0
    9 Power_On_Hours          0x0032   001   001   000    Old_age   Always       -       77571
  10 Spin_Retry_Count        0x0012   100   253   000    Old_age   Always       -       0
  11 Calibration_Retry_Count 0x0012   100   253   000    Old_age   Always       -       0
  12 Power_Cycle_Count       0x0032   100   100   000    Old_age   Always       -       52
  192 Power-Off_Retract_Count 0x0032   200   200   000    Old_age   Always       -       50
  193 Load_Cycle_Count        0x0032   200   200   000    Old_age   Always       -       52
  194 Temperature_Celsius     0x0022   117   102   000    Old_age   Always       -       30
  196 Reallocated_Event_Count 0x0032   200   200   000    Old_age   Always       -       0
  197 Current_Pending_Sector  0x0012   200   200   000    Old_age   Always       -       0
  198 Offline_Uncorrectable   0x0010   200   200   000    Old_age   Offline      -       0
  200 Multi_Zone_Error_Rate   0x0008   200   200   000    Old_age   Offline      -       0
  ~~~

* With SAS disks check the read (and write) **Total Uncorrected Errors** and **Elements in grown defect list**. If non zero, the disk is breaking.
  * Source: <https://www.ixsystems.com/community/threads/understanding-smartctl-sas.54754/>

  ~~~sh
  grep -e Power_On_Hours smartctl_a_sg*
  smartctl_a_sg10:  9 Power_On_Hours          0x0032   099   099   000    Old_age   Always       -       972
  smartctl_a_sg5:  9 Power_On_Hours          0x0032   059   059   000    Old_age   Always       -       30301
  smartctl_a_sg6:  9 Power_On_Hours          0x0012   089   089   000    Old_age   Always       -       82228
  smartctl_a_sg9:  9 Power_On_Hours          0x0032   032   032   000    Old_age   Always       -       59597
  ~~~

* For loop to get all disks **smartctl -a output**
  * `for disk in {4..11}; do sudo smartctl -a -d sat /dev/sg$disk > /tmp/smartctl_a_sg$disk; done`

* One liner for SAS disks for analyzing outputs
  * `grep "Elements in grown defect list\|Failed in segment" ./smartctl_a_sg* > smartctl_elements_in_grown_defect_list`

* Script for analyzing **smartctl -a** output of SATA disks
  * This doesn't work with SAS disks
  * Change /tmp/smartctl_a_sg* if needed.

  ~~~sh
  files=/tmp/smartctl_a_sg*; \
  output_smartctl=($(grep \
      -e Reallocated_Sector_Ct \
      -e Spin_Retry_Count \
      -e Reallocated_Event_Count \
      -e Current_Pending_Sector \
      -e Offline_Uncorrectable \
      $files \
      | sed 's/^smartctl_a_//' \
      | sed 's/: */:/' \
      | tr -s ' ' \
      | cut -d' ' -f1,10 \
      | sed 's/ /;/')); \
  echo "Printing all disks:"; \
  echo "Explanation:"; \
  echo "disk;error_ID;number_of_errors"; \
  printf '%s\n' ${output_smartctl[@]}; \
  for i in "${output_smartctl[@]}"; do
      #printf $i | cut -d';' -f2
      if [ $(printf $i | cut -d';' -f2) != 0 ]
      then
          echo "Faulty disk: $i"
      fi
  done <<< $(printf '%s\n' $output_smartctl); \
  echo "If there's no line starting 'Faulty disk:', probably all disks are fine."
  ~~~

* Example output
  * disk;error_ID;number_of_errors

  ~~~sh
  Printing all disks:
  sg10:5;0
  sg10:10;0
  sg10:196;0
  sg10:197;0
  sg10:198;0
  sg5:5;0
  sg5:10;0
  sg5:196;0
  sg5:197;0
  sg5:198;0
  sg6:5;0
  sg6:10;0
  sg6:196;0
  sg6:197;0
  sg6:198;0
  sg9:5;14
  sg9:10;0
  sg9:197;0
  sg9:198;1
  Faulty disk: sg9:5;14
  Faulty disk: sg9:198;1
  ~~~

## Test USB disk

* List disks with `sudo fdisk -l`
* Start test `sudo smartctl -t short /dev/sdb`

## Testing disk write speed with Ubuntu 16.04 Live USB

* In this test there are 8 disks in RAID10 with Adaptec 5805 controller.
* Main source of info: <https://www.cyberciti.biz/faq/howto-linux-unix-test-disk-performance-with-dd-command/>

### Make partition for the new disk

~~~sh
ubuntu@ubuntu:~$ sudo parted /dev/sdb
GNU Parted 3.2
Using /dev/sdb
Welcome to GNU Parted! Type 'help' to view a list of commands.
(parted) print
Error: /dev/sdb: unrecognised disk label
Model: Adaptec RAID10 (scsi)
Disk /dev/sdb: 3999GB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags:
(parted) mklabel gpt
(parted) mkpart
Partition name?  []?
File system type?  [ext2]? ext4
Start? 0%
End? 100%
(parted) print
Model: Adaptec RAID10 (scsi)
Disk /dev/sdb: 3999GB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name  Flags
 1      1049kB  3999GB  3999GB  ext4

(parted) quit
Information: You may need to update /etc/fstab.

ubuntu@ubuntu:~$ ls /dev/sd*
/dev/sda  /dev/sdb  /dev/sdb1  /dev/sdc  /dev/sdc1
~~~

### Format the disk

~~~sh
ubuntu@ubuntu:~$ sudo mkfs.ext4 /dev/sdb1
mke2fs 1.42.13 (17-May-2015)
Creating filesystem with 976223744 4k blocks and 244056064 inodes
Filesystem UUID: 5c473803-1bea-4aeb-9144-4353de1d3e21
Superblock backups stored on blocks:
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,
        102400000, 214990848, 512000000, 550731776, 644972544

Allocating group tables: done
Writing inode tables: done
Creating journal (32768 blocks): done
Writing superblocks and filesystem accounting information: done
~~~

### Create mount point and mount

~~~sh
ubuntu@ubuntu:~$ sudo mkdir /mnt/sdb1
ubuntu@ubuntu:~$ sudo mount /dev/sdb1  /mnt/sdb1/
~~~

### Test speed of RAID10 with SATA 8x WD RED 1TB HDDs, no caches

* All caches are disabled as the battery unit is super old.

~~~sh
# The first test is fast
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sdb1/somefile bs=1M count=4096
4096+0 records in
4096+0 records out
4294967296 bytes (4.3 GB, 4.0 GiB) copied, 3.4326 s, 1.3 GB/s

# Repeating the test speed drops to around 330-340 MB/s
sudo dd if=/dev/zero of=/mnt/sdb1/somefile bs=1M count=4096
4096+0 records in
4096+0 records out
4294967296 bytes (4.3 GB, 4.0 GiB) copied, 12.8237 s, 335 MB/s

# Big file
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sdb1/somefile2 bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 3.89733 s, 276 MB/s

# Read test for the big file
ubuntu@ubuntu:~$ time dd if=/mnt/sdb1/somefile2 of=/dev/null bs=8k
131072+0 records in
131072+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 8.23336 s, 130 MB/s

real    0m8.267s
user    0m0.028s
sys     0m0.668s



ubuntu@ubuntu:~$ sudo hdparm -tT /dev/sdb
/dev/sdb:
 Timing cached reads:   14064 MB in  2.00 seconds = 7037.92 MB/sec
 Timing buffered disk reads: 714 MB in  3.00 seconds = 237.76 MB/sec
 
ubuntu@ubuntu:~$ sudo hdparm -tT /dev/sdb
/dev/sdb:
 Timing cached reads:   13728 MB in  2.00 seconds = 6869.65 MB/sec
 Timing buffered disk reads: 716 MB in  3.00 seconds = 238.53 MB/sec
 
ubuntu@ubuntu:~$ sudo hdparm -tT /dev/sdb
/dev/sdb:
 Timing cached reads:   13808 MB in  2.00 seconds = 6909.67 MB/sec
 Timing buffered disk reads: 754 MB in  3.01 seconds = 250.55 MB/sec



ubuntu@ubuntu:~$ dmesg | grep -i sata | grep 'link up'
[    7.429059] ata1.00: SATA link up 1.5 Gbps (SStatus 113 SControl 300)


# Finding server latency time
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sdb1/test2.img bs=512 count=1000 oflag=dsync
1000+0 records in
1000+0 records out
512000 bytes (512 kB, 500 KiB) copied, 2.81873 s, 182 kB/s
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sdb1/test2.img bs=512 count=1000 oflag=dsync
1000+0 records in
1000+0 records out
512000 bytes (512 kB, 500 KiB) copied, 2.40865 s, 213 kB/s
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sdb1/test2.img bs=512 count=1000 oflag=dsync
1000+0 records in
1000+0 records out
512000 bytes (512 kB, 500 KiB) copied, 2.07194 s, 247 kB/s
~~~

### Test speed of single SSD in same hardware

~~~sh
# Small file written multiple times
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/somefile bs=1M count=4096
4096+0 records in
4096+0 records out
4294967296 bytes (4.3 GB, 4.0 GiB) copied, 3.44472 s, 1.2 GB/s
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/somefile bs=1M count=4096
4096+0 records in
4096+0 records out
4294967296 bytes (4.3 GB, 4.0 GiB) copied, 21.2597 s, 202 MB/s
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/somefile bs=1M count=4096
4096+0 records in
4096+0 records out
4294967296 bytes (4.3 GB, 4.0 GiB) copied, 21.295 s, 202 MB/s






# Big file test
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/somefile2 bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 1.1785 s, 911 MB/s
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/somefile2 bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 5.29935 s, 203 MB/s
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/somefile2 bs=1G count=1
1+0 records in
1+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 5.7742 s, 186 MB/s

# Read test for the big file
ubuntu@ubuntu:~$ time dd if=/mnt/sda1/somefile2 of=/dev/null bs=8k
131072+0 records in
131072+0 records out
1073741824 bytes (1.1 GB, 1.0 GiB) copied, 0.266076 s, 4.0 GB/s

real    0m0.268s
user    0m0.016s
sys     0m0.248s



ubuntu@ubuntu:~$ sudo hdparm -tT /dev/sda
/dev/sda:
 Timing cached reads:   14040 MB in  2.00 seconds = 7025.92 MB/sec
 Timing buffered disk reads: 764 MB in  3.01 seconds = 254.08 MB/sec

ubuntu@ubuntu:~$ sudo hdparm -tT /dev/sda
/dev/sda:
 Timing cached reads:   13814 MB in  2.00 seconds = 6913.20 MB/sec
 Timing buffered disk reads: 764 MB in  3.01 seconds = 254.16 MB/sec

ubuntu@ubuntu:~$ sudo hdparm -tT /dev/sda
/dev/sda:
 Timing cached reads:   13826 MB in  2.00 seconds = 6919.75 MB/sec
 Timing buffered disk reads: 764 MB in  3.01 seconds = 254.17 MB/sec

# Finding server latency time
# Seems that the latency is bettter with SSD than HDD RAID
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/test2.img bs=512 count=1000 oflag=dsync
1000+0 records in
1000+0 records out
512000 bytes (512 kB, 500 KiB) copied, 0.801241 s, 639 kB/s
ubuntu@ubuntu:~$ sudo dd if=/dev/zero of=/mnt/sda1/test2.img bs=512 count=1000 oflag=dsync
1000+0 records in
1000+0 records out
512000 bytes (512 kB, 500 KiB) copied, 0.810221 s, 632 kB/s
~~~
