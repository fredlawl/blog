+++
date = '2025-05-25T14:08:02-05:00'
draft = true
title = 'journalctl BTRFS csum corruption'
toc = true
categories = ["debuging"]
tags = ["gdb", "os", "systemd", "btrfs"]
next = true
+++

## Background

Shortly after installing Fedora (Kernel 6.14.6-300.fc42.x86_64) on a freshly
wiped disk (old Windows NTFS games/extra disk), I noticed that `journalctl`
then `<shift>-G` would result in a `journalctl` crash for `systemd 257 (257.5-6.fc42)` :

```bash
➜  ~ coredumpctl info
           PID: 11701 (journalctl)
           UID: 1000 (fred)
           GID: 1000 (fred)
        Signal: 6 (ABRT)
     Timestamp: Sun 2025-05-25 10:19:33 CDT (40min ago)
  Command Line: journalctl -xe
    Executable: /usr/bin/journalctl
 Control Group: /user.slice/user-1000.slice/user@1000.service/app.slice/app-gnome-Alacritty-6480.scope
          Unit: user@1000.service
     User Unit: app-gnome-Alacritty-6480.scope
         Slice: user-1000.slice
     Owner UID: 1000 (fred)
       Boot ID: 81b2c199435745759afa8439492049e5
    Machine ID: 42148a0b9b9641b0ab3c18373236fe58
      Hostname: olympus
       Storage: /var/lib/systemd/coredump/core.journalctl.1000.81b2c199435745759afa8439492049e5.11701.1748186373000000.zst (present)
  Size on Disk: 183.3K
       Package: systemd/257.5-6.fc42
      build-id: 812a15616de1baca7c7c2942d40b5dbb73c6b905
       Message: Process 11701 (journalctl) of user 1000 dumped core.

                Module /usr/bin/journalctl from rpm systemd-257.5-6.fc42.x86_64
                Module libzstd.so.1 from rpm zstd-1.5.7-1.fc42.x86_64
                Module libcap-ng.so.0 from rpm libcap-ng-0.8.5-4.fc42.x86_64
                Module libpcre2-8.so.0 from rpm pcre2-10.45-1.fc42.x86_64
                Module libeconf.so.0 from rpm libeconf-0.7.6-1.fc42.x86_64
                Module libaudit.so.1 from rpm audit-4.0.3-2.fc42.x86_64
                Module libz.so.1 from rpm zlib-ng-2.2.4-3.fc42.x86_64
                Module libattr.so.1 from rpm attr-2.5.2-5.fc42.x86_64
                Module libselinux.so.1 from rpm libselinux-3.8-1.fc42.x86_64
                Module libseccomp.so.2 from rpm libseccomp-2.5.5-2.fc41.x86_64
                Module libpam.so.0 from rpm pam-1.7.0-5.fc42.x86_64
                Module libcrypto.so.3 from rpm openssl-3.2.4-3.fc42.x86_64
                Module libmount.so.1 from rpm util-linux-2.40.4-7.fc42.x86_64
                Module libcrypt.so.2 from rpm libxcrypt-4.4.38-7.fc42.x86_64
                Module libcap.so.2 from rpm libcap-2.73-2.fc42.x86_64
                Module libblkid.so.1 from rpm util-linux-2.40.4-7.fc42.x86_64
                Module libacl.so.1 from rpm acl-2.3.2-3.fc42.x86_64
                Module libsystemd-shared-257.5-6.fc42.so from rpm systemd-257.5-6.fc42.x86_64
                Stack trace of thread 11701:
                #0  0x00007f3ec948111c __pthread_kill_implementation (libc.so.6 + 0x7311c)
                #1  0x00007f3ec9427afe raise (libc.so.6 + 0x19afe)
                #2  0x00007f3ec940f6d0 abort (libc.so.6 + 0x16d0)
                #3  0x00007f3ec983b2cc mmap_cache_process_sigbus (libsystemd-shared-257.5-6.fc42.so + 0x23b2cc)
                #4  0x00007f3ec983b5bf mmap_cache_fd_free (libsystemd-shared-257.5-6.fc42.so + 0x23b5bf)
                #5  0x00007f3ec98274fc journal_file_close (libsystemd-shared-257.5-6.fc42.so + 0x2274fc)
                #6  0x00007f3ec9845150 sd_journal_close (libsystemd-shared-257.5-6.fc42.so + 0x245150)
                #7  0x000055db6237d769 run (/usr/bin/journalctl + 0x7769)
                #8  0x000055db62378145 main (/usr/bin/journalctl + 0x2145)
                #9  0x00007f3ec94115f5 __libc_start_call_main (libc.so.6 + 0x35f5)
                #10 0x00007f3ec94116a8 __libc_start_main@@GLIBC_2.34 (libc.so.6 + 0x36a8)
                #11 0x000055db623783c5 _start (/usr/bin/journalctl + 0x23c5)
                ELF object binary architecture: AMD x86-64
```

Looking at `sudo dmesg` (Kernel logs) I saw several:

```bash
➜  ~ sudo dmesg
...
[ 2614.020827] BTRFS warning (device sda3): csum failed root 256 ino 386149 off 0 csum 0x8941f998 expected csum 0x33a303fe mirror 1
[ 2614.020828] BTRFS error (device sda3): bdev /dev/sda3 errs: wr 0, rd 0, flush 0, corrupt 414416, gen 0
[ 2614.020828] BTRFS warning (device sda3): csum failed root 256 ino 386149 off 4096 csum 0x8941f998 expected csum 0x3e2e9220 mirror 1
[ 2614.020829] BTRFS error (device sda3): bdev /dev/sda3 errs: wr 0, rd 0, flush 0, corrupt 414417, gen 0
```

I am working on a few systemd problems with upstream on their latest
v257 stable release, and thought there may be another systemd problem here.
I know better to not report right away, so I needed to figure out if there
is a disk or filesystem problem first, and then look for solutions.

## Tools

I searched the internet for clues on what to-do. I found the following to help
debug the issue:

1. `smartctl` A tool to diagnose disk HW problems
2. `btrfs scrub` A tool to help find/repair corruptions (if able)
3. `btrfs check` A tool to repair BTRFS file system (WARNING! AVOID USE)
4. `btrfs inspect-internal` A tool to check information about parts of the disk/filesystem

## Investigation

Starting with HW specific check:

```bash
➜  ~ sudo smartctl -x /dev/sda3
smartctl 7.5 2025-04-30 r5714 [x86_64-linux-6.14.6-300.fc42.x86_64] (local build)
Copyright (C) 2002-25, Bruce Allen, Christian Franke, www.smartmontools.org

=== START OF INFORMATION SECTION ===
Model Family:     Samsung based SSDs
Device Model:     Samsung SSD 860 EVO 250GB
Serial Number:    S3YHNX0K415898P
LU WWN Device Id: 5 002538 e40306b00
Firmware Version: RVT01B6Q
User Capacity:    250,059,350,016 bytes [250 GB]
Sector Size:      512 bytes logical/physical
Rotation Rate:    Solid State Device
Form Factor:      2.5 inches
TRIM Command:     Available, deterministic, zeroed
Device is:        In smartctl database 7.5/5706
ATA Version is:   ACS-4 T13/BSR INCITS 529 revision 5
SATA Version is:  SATA 3.1, 6.0 Gb/s (current: 6.0 Gb/s)
Local Time is:    Sun May 25 11:11:34 2025 CDT
SMART support is: Available - device has SMART capability.
SMART support is: Enabled
AAM feature is:   Unavailable
APM feature is:   Unavailable
Rd look-ahead is: Enabled
Write cache is:   Enabled
DSN feature is:   Unavailable
ATA Security is:  Disabled, frozen [SEC2]

=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED

General SMART Values:
Offline data collection status:  (0x00) Offline data collection activity
     was never started.
     Auto Offline Data Collection: Disabled.
Self-test execution status:      (   0) The previous self-test routine completed
     without error or no self-test has ever
     been run.
Total time to complete Offline
data collection:   (    0) seconds.
Offline data collection
capabilities:     (0x53) SMART execute Offline immediate.
     Auto Offline data collection on/off support.
     Suspend Offline collection upon new
     command.
     No Offline surface scan supported.
     Self-test supported.
     No Conveyance Self-test supported.
     Selective Self-test supported.
SMART capabilities:            (0x0003) Saves SMART data before entering
     power-saving mode.
     Supports SMART auto save timer.
Error logging capability:        (0x01) Error logging supported.
     General Purpose Logging supported.
Short self-test routine
recommended polling time:   (   2) minutes.
Extended self-test routine
recommended polling time:   (  85) minutes.
SCT capabilities:         (0x003d) SCT Status supported.
     SCT Error Recovery Control supported.
     SCT Feature Control supported.
     SCT Data Table supported.

SMART Attributes Data Structure revision number: 1
Vendor Specific SMART Attributes with Thresholds:
ID# ATTRIBUTE_NAME          FLAGS    VALUE WORST THRESH FAIL RAW_VALUE
  5 Reallocated_Sector_Ct   PO--CK   100   100   010    -    0
  9 Power_On_Hours          -O--CK   099   099   000    -    4684
 12 Power_Cycle_Count       -O--CK   099   099   000    -    959
177 Wear_Leveling_Count     PO--C-   099   099   000    -    6
179 Used_Rsvd_Blk_Cnt_Tot   PO--C-   100   100   010    -    0
181 Program_Fail_Cnt_Total  -O--CK   100   100   010    -    0
182 Erase_Fail_Count_Total  -O--CK   100   100   010    -    0
183 Runtime_Bad_Block       PO--C-   100   100   010    -    0
187 Uncorrectable_Error_Cnt -O--CK   100   100   000    -    0
190 Airflow_Temperature_Cel -O--CK   076   058   000    -    24
195 ECC_Error_Rate          -O-RC-   200   200   000    -    0
199 CRC_Error_Count         -OSRCK   100   100   000    -    0
235 POR_Recovery_Count      -O--C-   099   099   000    -    42
241 Total_LBAs_Written      -O--CK   099   099   000    -    2043498859
                            ||||||_ K auto-keep
                            |||||__ C event count
                            ||||___ R error rate
                            |||____ S speed/performance
                            ||_____ O updated online
                            |______ P prefailure warning

General Purpose Log Directory Version 1
SMART           Log Directory Version 1 [multi-sector log support]
Address    Access  R/W   Size  Description
0x00       GPL,SL  R/O      1  Log Directory
0x01           SL  R/O      1  Summary SMART error log
0x02           SL  R/O      1  Comprehensive SMART error log
0x03       GPL     R/O      1  Ext. Comprehensive SMART error log
0x04       GPL,SL  R/O      8  Device Statistics log
0x06           SL  R/O      1  SMART self-test log
0x07       GPL     R/O      1  Extended self-test log
0x09           SL  R/W      1  Selective self-test log
0x10       GPL     R/O      1  NCQ Command Error log
0x11       GPL     R/O      1  SATA Phy Event Counters log
0x13       GPL     R/O      1  SATA NCQ Send and Receive log
0x30       GPL,SL  R/O      9  IDENTIFY DEVICE data log
0x80-0x9f  GPL,SL  R/W     16  Host vendor specific log
0xa1           SL  VS      16  Device vendor specific log
0xa5           SL  VS      16  Device vendor specific log
0xce-0xcf      SL  VS      16  Device vendor specific log
0xe0       GPL,SL  R/W      1  SCT Command/Status
0xe1       GPL,SL  R/W      1  SCT Data Transfer

SMART Extended Comprehensive Error Log Version: 1 (1 sectors)
No Errors Logged

SMART Extended Self-test Log Version: 1 (1 sectors)
No self-tests have been logged.  [To run self-tests, use: smartctl -t]

SMART Selective self-test log data structure revision number 1
 SPAN  MIN_LBA  MAX_LBA  CURRENT_TEST_STATUS
    1        0        0  Not_testing
    2        0        0  Not_testing
    3        0        0  Not_testing
    4        0        0  Not_testing
    5        0        0  Not_testing
Selective self-test flags (0x0):
  After scanning selected spans, do NOT read-scan remainder of disk.
If Selective self-test is pending on power-up, resume after 0 minute delay.

SCT Status Version:                  3
SCT Version (vendor specific):       256 (0x0100)
Device State:                        Active (0)
Current Temperature:                    24 Celsius
Power Cycle Min/Max Temperature:     21/40 Celsius
Lifetime    Min/Max Temperature:     17/42 Celsius
Specified Max Operating Temperature:    55 Celsius
Under/Over Temperature Limit Count:   0/0
SMART Status:                        0xc24f (PASSED)

SCT Temperature History Version:     2
Temperature Sampling Period:         1 minute
Temperature Logging Interval:        10 minutes
Min/Max recommended Temperature:      0/70 Celsius
Min/Max Temperature Limit:            0/70 Celsius
Temperature History Size (Index):    128 (49)

Index    Estimated Time   Temperature Celsius
  50    2025-05-24 14:00    27  ********
[ snip ]
  48    2025-05-25 11:00    23  ****
  49    2025-05-25 11:10    24  *****

SCT Error Recovery Control:
           Read: Disabled
          Write: Disabled

Device Statistics (GP Log 0x04)
Page  Offset Size        Value Flags Description
0x01  =====  =               =  ===  == General Statistics (rev 1) ==
0x01  0x008  4             959  ---  Lifetime Power-On Resets
0x01  0x010  4            4684  ---  Power-on Hours
0x01  0x018  6      2043498859  ---  Logical Sectors Written
0x01  0x020  6        12091242  ---  Number of Write Commands
0x01  0x028  6      1406882279  ---  Logical Sectors Read
0x01  0x030  6        16026334  ---  Number of Read Commands
0x01  0x038  6         2135000  ---  Date and Time TimeStamp
0x04  =====  =               =  ===  == General Errors Statistics (rev 1) ==
0x04  0x008  4               0  ---  Number of Reported Uncorrectable Errors
0x04  0x010  4               0  ---  Resets Between Cmd Acceptance and Completion
0x05  =====  =               =  ===  == Temperature Statistics (rev 1) ==
0x05  0x008  1              24  ---  Current Temperature
0x05  0x020  1              42  ---  Highest Temperature
0x05  0x028  1              17  ---  Lowest Temperature
0x05  0x058  1              55  ---  Specified Maximum Operating Temperature
0x06  =====  =               =  ===  == Transport Statistics (rev 1) ==
0x06  0x008  4            4191  ---  Number of Hardware Resets
0x06  0x010  4               0  ---  Number of ASR Events
0x06  0x018  4               0  ---  Number of Interface CRC Errors
0x07  =====  =               =  ===  == Solid State Device Statistics (rev 1) ==
0x07  0x008  1               0  N--  Percentage Used Endurance Indicator
                                |||_ C monitored condition met
                                ||__ D supports DSN
                                |___ N normalized value

Pending Defects log (GP Log 0x0c) not supported

SATA Phy Event Counters (GP Log 0x11)
ID      Size     Value  Description
0x0001  2            0  Command failed due to ICRC error
0x0002  2            0  R_ERR response for data FIS
0x0003  2            0  R_ERR response for device-to-host data FIS
0x0004  2            0  R_ERR response for host-to-device data FIS
0x0005  2            0  R_ERR response for non-data FIS
0x0006  2            0  R_ERR response for device-to-host non-data FIS
0x0007  2            0  R_ERR response for host-to-device non-data FIS
0x0008  2            0  Device-to-host non-data FIS retries
0x0009  2         1994  Transition from drive PhyRdy to drive PhyNRdy
0x000a  2            2  Device-to-host register FISes sent due to a COMRESET
0x000b  2            0  CRC errors within host-to-device FIS
0x000d  2            0  Non-CRC errors within host-to-device FIS
0x000f  2            0  R_ERR response for host-to-device data FIS, CRC
0x0010  2            0  R_ERR response for host-to-device data FIS, non-CRC
0x0012  2            0  R_ERR response for host-to-device non-data FIS, CRC
0x0013  2            0  R_ERR response for host-to-device non-data FIS, non-CRC
```

There is a lot here! But the gist is:

```bash
=== START OF READ SMART DATA SECTION ===
SMART overall-health self-assessment test result: PASSED
...
SATA Phy Event Counters (GP Log 0x11)
ID      Size     Value  Description
0x0001  2            0  Command failed due to ICRC error
0x0002  2            0  R_ERR response for data FIS
0x0003  2            0  R_ERR response for device-to-host data FIS
0x0004  2            0  R_ERR response for host-to-device data FIS
0x0005  2            0  R_ERR response for non-data FIS
0x0006  2            0  R_ERR response for device-to-host non-data FIS
0x0007  2            0  R_ERR response for host-to-device non-data FIS
0x0008  2            0  Device-to-host non-data FIS retries
0x0009  2         1994  Transition from drive PhyRdy to drive PhyNRdy
0x000a  2            2  Device-to-host register FISes sent due to a COMRESET
0x000b  2            0  CRC errors within host-to-device FIS
0x000d  2            0  Non-CRC errors within host-to-device FIS
0x000f  2            0  R_ERR response for host-to-device data FIS, CRC
0x0010  2            0  R_ERR response for host-to-device data FIS, non-CRC
0x0012  2            0  R_ERR response for host-to-device non-data FIS, CRC
0x0013  2            0  R_ERR response for host-to-device non-data FIS, non-CRC
```

Which tells me the SSD is likely fine. This gave me a sigh of relief. Next
I checked to see if btrfs can repair itself:

```bash
➜  ~ sudo btrfs scrub start /
➜  ~ sudo dmesg -w
...
[ 3136.872126] BTRFS info (device sda3): scrub: started on devid 1
[ 3159.503387] BTRFS error (device sda3): unable to fixup (regular) error at logical 12463439872 on dev /dev/sda3 physical 13545570304
[ 3159.503415] BTRFS error (device sda3): unable to fixup (regular) error at logical 12463439872 on dev /dev/sda3 physical 13545570304
[ 3168.222718] BTRFS info (device sda3): scrub: finished on devid 1 with status: 0
```

Still an error, but un-fixiable... So next I ran:

> WARNING: DO NOT RUN THE FOLLOWING WITHOUT --readonly on a mounted device!

```bash
sudo btrfs check --force --readonly --check-data-csum /dev/sda3
```

I got more of the same information that I already knew. It occurred
to me at this point there was a critical clue all along:

```bash
... csum failed root 256 ino 386149 off 0 ...
~~~~~~~~~~~~~~~~~~~~~~~~~^
```

These messages were all for a single inode! Performing a lookup on that,
I found the culprit:

```bash
➜  ~ sudo btrfs inspect-internal inode-resolve 386149 /
//var/lib/systemd/catalog/database
```

## Solution

This confirms that this problem is very much localized to systemd! The next
question is how to fix? I first thought to delete the file, and
hope systemd will remake it. But I wasn't sure. I knew through
searching this file was directly tied to the systemd catalog, and I found
a journal command to use via `man 1 journalctl` :

```bash
       --update-catalog
           Update the message catalog index. This command needs to be
           executed each time new catalog files are installed, removed, or
           updated to rebuild the binary catalog index.

           Added in version 196.
```

I thought, why not... I ran `sudo journalctl --update-catalog` and now
the problem is fixed!

It turns out the problem was a *me* problem all along. If that didn't
resolve the problem, I was going to delete the file manually, and
continue the investigation and attempt to reproduce the problem with
systemd + btrfs filesystem.

To be fair, I installed Fedora freshly just last week, and I had
*some* issues with the install process. It could've happened
at that time. I still don't know what the actual trigger is. I can only
speculate some interruption in btrfs during the initial creation process.

## Lesson learned

At first thought, the main lesson here is to study these error messages
in more detail. The same inode repeatedly shows up in logs, and thus
that provides a good starting point.

However, inodes can constantly change. If this file was removed/created, by
the time I check, it's no longer a valid inode number. I'd be back at
square one. Secondly, this could've been a much bigger problem ranging
from a bug in the filesystem code all the way to a HW failure. It was good I
still went through the steps to rule out those possibilities.

All that aside, I'm just happy my disk is fine and I can read output
from `journalctl` again to debug other things 😂
