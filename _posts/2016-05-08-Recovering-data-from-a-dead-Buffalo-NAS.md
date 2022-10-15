---
title: Recovering data from a dead Buffalo NAS
---
#### I take a look at what happened with my Buffalo NAS, and how to recover data from its dead corpse.

A few years ago, when everything was getting more and more connected, I decided to buy a NAS. It shouldn't be expensive, so I got a Buffalo LS-WXL. It had two 2TB drives, and some seriously shitty hardware specs, but it was enough at that time. Not beeing completely stupid, I set it up to back up the first drive to the second one every week (which I thanked myself for later).

But then, two months ago, it started to email me write errors on one of the hard drives (the one I backed up to). I thought "A broken hard drive, especially a desktop HDD, that can happen" and wanted to replace it. That was when the NAS turned off (or hung, whatever). I got it back online, and copied all the stuff to another drive. The next morning, the NAS was off again. And it didn't turn on properly again, so I took it off the network, removed the drives and attached them to a PC.   
It was fairly easy to copy all the data off since it wasn't striped (RAID 0) (There is a great guide on how to do that at [skelleton.net](https://www.skelleton.net/2014/04/06/rescuing-data-from-a-buffalo-link-station-with-failed-a-raid/), and you should probably read it first anyway). The file system Buffalo uses is XFS, the proper drivers are included in `xfs-progs` on Linux. When looking at the partition layout of the drive with `lsblk`, you will see six partitions on the drive, with the last one having a Linux MD RAID 1 in it.  On the right, the name of the array, for example `md123` is shown. You should then be able to then mount read-only (very much recommended!) it with `mount -o ro /dev/md123 /mnt/hdd`.  
You can then copy it off using either `rsync` or just plain ol' `cp` like this:  

    cp -av /mnt/hdd/* /backuplocation

The `a` flag will preserve any modification dates on files and the `v` makes if verbose (shows what file is currently getting copied).  

So the data was safe, but what happened? When doing SMART-Tests (there are many guides on how to do this, I used GSmartControl to make it easy) of the first drive, everything was good:

~~~
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Short offline       Completed without error       00%     19206         -
~~~
  
Other values to look out for are `Reallocated_Sector_Ct`,  `Offline_Uncorrectable` and `Current_Pending_Sector`. These will tell you how many bad sectors were reallocated, how many can not be reallocated and how many are still pending to be reallocated. If you get a few on the first value, that is mostly fine. Sectors do die. For the first drive, everything was okay again:

~~~
5   Reallocated_Sector_Ct   0x0033   100   100   036    Pre-fail  Always       -       0
197 Current_Pending_Sector  0x0012   100   100   000    Old_age   Always       -       0
198 Offline_Uncorrectable   0x0010   100   100   000    Old_age   Offline      -       0
~~~

Another value that caught my attention was this one:

~~~
190 Airflow_Temperature_Cel 0x0022   071   043   045    Old_age   Always   In_the_past 29 (0 4 29 27 0)
~~~

The hard drive got too hot in the NAS, which can significantly reduce it's lifespan.

But the other drive (the one I backed up to) was way more interesting:

~~~
Num  Test_Description    Status                  Remaining  LifeTime(hours)  LBA_of_first_error
# 1  Extended offline    Completed: read failure       90%     19205         62888117
# 2  Short offline       Completed: read failure       90%     19205         62888117
~~~

That doesn't look good at all. If we take a look at the previously mentioned values:

~~~
5   Reallocated_Sector_Ct   0x0033   051   051   036    Pre-fail  Always       -       64800
197 Current_Pending_Sector  0x0012   100   100   000    Old_age   Always       -       96
198 Offline_Uncorrectable   0x0010   100   100   000    Old_age   Offline      -       96
~~~

There are tutorials on how to fix single uncorrectable sectors by zeroing them out with `dd`, but 96 are *way* too many. Seeing the 64800 reallocated sectors, the hard drive probably ran out of spare sectors. I decided to crap that drive.

But what happened to the NAS itself? The truth is, I don't know. Maybe it didn't want to boot with all the drive errors, but considering that it doesn't run SMART tests, I would rather bet on it crapping it's pants at the same time as the hard drive.
