# The Simple Guide to Btrfs
Deep Grewal - November 25, 2020

___
## What is Btrfs?
Btrfs (b-tree filesystem) is commonly pronounced as "Better F-S", "Butter F-S", and "B-tree F-S".  Btrfs is a filesystem that was designed at Oracle in 2007 and has since been in use throughout the Linux community.  By 2012, Oracle Linux and SUSE Linux Enterprise had adopted Btrfs as a production-viable and supported file system.  In November of 2013, the filesystem was considered to be stable by the Linux community and was officially incorporated into the Linux kernel.  SUSE Linux Enterprise Server 12 was the first distribution of Linux to make Btrfs the default filesystem in 2015.  Fedora 33 has been the most recent distribution to do the same in 2020.  As time progresses, perhaps more distributions will do the same and adopt Btrfs as a default as well.

### Under the Hood
Now that we have some history established, let's have a peek at the technical underpinnings. Btrfs uses the copy-on-write (COW) methodology of writing information to disks.  In addition to COW, Btrfs takes advantage of pooling, snapshots, and checksums. It is also capable of enabling RAID across multiple devices.  The remainder of this document will focus on subvolumes and snapshots.

#### Subvolumes
A Btrfs subvolume is similar to a separate POSIX file namespace.  Each subvolume can be mounted to its own unique mount-point.  Subvolumes are flexible.  They can be created at any location within the filesystem and also can be nested within other subvolumes.  Due to their heirarchical nature, a subvolume containing other nested/children subvolumes cannot be deleted until the nested/children subvolumes are deleted first.  Each implementation of Btrfs will contain a parent/main subvolume.  This is the top-level subvolume that is mounted by default.  To a user, subvolumes appear as directories and subdirectories.  

#### Snapshots
Snapshots are simply just Btrfs subvolumes.  A snapshot can be described as a separate subvolume that contains the metadata and data of another subvolume.  Since snapshots themselves are subvolumes, they too can be nested.  Essentially, the snapshot is a backup or redundant version of a subvolume.  The process of creating a snapshot is not recursive.  So, if a subvolume contains additional nested subvolumes, the data within those nested subvolumes will not be captured when a snapshot is created of the parent subvolume.  Rather, the snapshot will contain an empty directory for each nested subvolume.  To ensure that the data of each nested subvolume has been captured, a separate snapshot should be made for each nested subvolume.

Thanks to Btrfs and its COW methodology, snapshots occcupy relatively little disk space and can be created in little time.  The copy-on-write process of creating a snapshot writes the differential between the original subvolume and the snapshot.  Similar to taking a snapshot of a virtual machine, the snapshot itself will be smaller in size than the source.  After a snapshot has been created, it can be mounted like any subvolume to a mount-point.

___
## Implementing Btrfs
For the purposes of this example, I have created an Ubuntu Server 20.04 virtual machine which contains 2 disks: `/dev/sda` and `/dev/sdb`.  

### The Initial Layout
The disk `/dev/sda` is a 15GB disk that contains the root partition.  The disk `/dev/sdb` is a 10GB disk and was added as a separate disk drive to demonstrate the implementation of Btrfs.  Think of it as an additional drive you add to your machine to save data on.

```
deep@ubuntu-vm:~$ lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
loop0    7:0    0 67.8M  1 loop /snap/lxd/18150
loop1    7:1    0   55M  1 loop /snap/core18/1880
loop2    7:2    0 55.4M  1 loop /snap/core18/1932
loop3    7:3    0 71.3M  1 loop /snap/lxd/16099
loop4    7:4    0   31M  1 loop /snap/snapd/9721
loop5    7:5    0 29.9M  1 loop /snap/snapd/8542
sda      8:0    0   15G  0 disk
├─sda1   8:1    0    1M  0 part
└─sda2   8:2    0   15G  0 part /
sdb      8:16   0   10G  0 disk
sr0     11:0    1 1024M  0 rom
```

### Formatting for Btrfs
Before we can do anything with Btrfs, the first step is to format `/dev/sdb` with the Btrfs filesystem.  Note that the `-f` option can be used to force the overwrite of any existing filesystem.

<br/>

>***Note:** The command below can lead to data loss.  Do not perform this command on a disk drive which contains data that is needed and has not been backed up to a separate location.*

<br/>

```
deep@ubuntu-vm:~$ sudo mkfs.btrfs -f /dev/sdb
btrfs-progs v5.4.1
See http://btrfs.wiki.kernel.org for more information.

Label:              (null)
UUID:               8e6d7c89-d65a-478b-84de-5b40df9d0a02
Node size:          16384
Sector size:        4096
Filesystem size:    10.00GiB
Block group profiles:
  Data:             single            8.00MiB
  Metadata:         DUP             256.00MiB
  System:           DUP               8.00MiB
SSD detected:       no
Incompat features:  extref, skinny-metadata
Checksum:           crc32c
Number of devices:  1
Devices:
   ID        SIZE  PATH
    1    10.00GiB  /dev/sdb
```

### Creating a Mount-Point and Mounting to the Mount-Point
Since `/dev/sdb` is intended to store additional data, we can create a mount-point at `/mnt/data` and mount `/dev/sdb` to this location.

First, let's create the mount-point.

```
deep@ubuntu-vm:~$ sudo mkdir -p /mnt/data
```

Now that the mount-point has been created, we can mount `/dev/sdb` to the newly created mount-point at `/mnt/data`.

```
deep@ubuntu-vm:~$ sudo mount -t btrfs /dev/sdb /mnt/data
```

#### Verifying a Successful Mount
The outputs of both of those last commands were pretty uneventful.  Let's run some checks just to make sure that all went as expected.  

Let's start by listing all of the block devices and filtering for `sdb`.  The output does indeed show that `sdb` is mounted at `/mnt/data`.

```
deep@ubuntu-vm:~$ lsblk | grep sdb
sdb      8:16   0   10G  0 disk /mnt/data
```

Now, let's try another command to list all of the mounts and filter for `sdb`.  The output of this command also shows a successful mount has occurred.  It also displays additional information about the btrfs subvolume.  Notably, that the subvolume has an id of 5 (`subvolid=5`) and that it is the parent or top-level subvolume (`subvol=/`).

```
deep@ubuntu-vm:~$ mount -l | grep sdb
/dev/sdb on /mnt/data type btrfs (rw,relatime,space_cache,subvolid=5,subvol=/)
```

And for good measure, let's try yet another method to verify that all went well by using the `btrfs subvolume show` command on the `/mnt/data` location.  The output of this command also indicates the subvolume id of 5 (`Subvolume ID: 5`).

```
deep@ubuntu-vm:~$ sudo btrfs subvolume show /mnt/data
/
        Name:                   <FS_TREE>
        UUID:                   a50cdf77-58cd-4762-9ad0-87a827192cbd
        Parent UUID:            -
        Received UUID:          -
        Creation time:          2020-11-25 17:57:34 +0000
        Subvolume ID:           5
        Generation:             4
        Gen at creation:        0
        Parent ID:              0
        Top level ID:           0
        Flags:                  -
        Snapshot(s):
```

<br/>

>***Note:** The default subvolume Id given to the root-level (highest-level) subvolume is 5.  Although a subvolume was not explicitly created, the act of mounting a Btrfs device created the root-level subvolume.*

<br/>

### Creating a Subvolume
Recall that `/dev/sdb` was intended to be an additional disk which we will use to store data on.  Let's extend this example and create another subvolume on this disk which will be used for storing documents.  We can call this new subvolume `documents`.  This subvolume will appear and behave as a subdirectory once it is created.  

```
deep@ubuntu-vm:~$ sudo btrfs subvolume create /mnt/data/documents
Create subvolume '/mnt/data/documents'
```

#### Verifying the Creation of a Subvolume
As before, the output of that last command was also pretty uneventful.  Let's run some checks to verify that we have a new subvolume.

Let's once again use the `btrfs subvolume show` command on the `/mnt/data/documents` location.  The output of this command indicates the subvolume id of 257 (`Subvolume ID: 257`).

```
deep@ubuntu-vm:~$ sudo btrfs subvolume show /mnt/data/documents
documents
        Name:                   documents
        UUID:                   82224209-acdb-fc46-b2af-daeff385d841
        Parent UUID:            -
        Received UUID:          -
        Creation time:          2020-11-25 18:37:03 +0000
        Subvolume ID:           257
        Generation:             7
        Gen at creation:        7
        Parent ID:              5
        Top level ID:           5
        Flags:                  -
        Snapshot(s):
```

Now is a good time to introduce another command that allows us to verify a subvolume.  The output of this command also indicates the subvolume id of 257 (`ID 257`).

```
deep@ubuntu-vm:~$ sudo btrfs subvolume list /mnt/data/documents
ID 257 gen 7 top level 5 path documents
```

### Generating Data for Snapshotting
In order to demonstrate snapshots, we should create some subdirectories and files that are "important" and in need of being protected and preserved by the Btrfs filesystem.

First, let's navigate to the `/mnt/data/documents` subvolume and create the `files` and `notes` subdirectories.

```
deep@ubuntu-vm:~$ cd /mnt/data/documents/

deep@ubuntu-vm:/mnt/data/documents$ sudo mkdir files notes

deep@ubuntu-vm:/mnt/data/documents$ ls
files  notes
```

Next, let's create some files within each of these subdirectories and verify their creation.

```
deep@ubuntu-vm:/mnt/data/documents$ sudo touch files/file{1..3}.txt

deep@ubuntu-vm:/mnt/data/documents$ sudo touch notes/note{1..3}.txt

deep@ubuntu-vm:/mnt/data/documents$ ls *
files:
file1.txt  file2.txt  file3.txt

notes:
note1.txt  note2.txt  note3.txt
```

### Creating a Snapshot
Everything is now in place to create a snapshot.  We now have: a Btrfs file system mounted, a subvolume created, and "important" data worthy of a snapshot.  

Let's create a snapshot for the subvolume `/mnt/data/documents` into the snapshot location `/mnt/data/documents/snapshots`.  If we wanted to make a read-only snapshot, we could provide the `-r` parameter to the `btrfs subvolume snapshots` command.  In this example, we will not make the snapshot read-only.

```
deep@ubuntu-vm:/mnt/data/documents$ sudo btrfs subvolume snapshot /mnt/data/documents /mnt/data/documents/snapshots
Create a snapshot of '/mnt/data/documents' in '/mnt/data/documents/snapshots'
```

#### Verifying the Creation of a Snapshot (Subvolume)
If you haven't yet noticed a pattern, by this point, you should note that some of the `btrfs` commands don't output too much fanfare when things are successfully created.  I suppose its fodder for a good written tutorial.

Let's verify the creation of the snapshot using a basic `ls` command.  The `-R` parameter allows us to recursively list the contents of each subdirectory.  The output of this command does indeed show that a `snapshots` subdirectory (subvolume) has been created which contains all of the data that was in the `/mnt/data/documents` source subvolume.

```
deep@ubuntu-vm:/mnt/data/documents$ ls -R *
files:
file1.txt  file2.txt  file3.txt

notes:
note1.txt  note2.txt  note3.txt

snapshots:
files  notes

snapshots/files:
file1.txt  file2.txt  file3.txt

snapshots/notes:
note1.txt  note2.txt  note3.txt
```

We should also verify using the `btrfs` commands.  The `btrfs subvolume show` command can be used on both subvolumes (the source subvolume and the snapshot).

When we run the command against the source subvolume the `Snapshot(s)` section does indeed state that there is an existance of snapshots in `documents/snapshots`.

```
deep@ubuntu-vm:/mnt/data/documents$ sudo btrfs subvolume show /mnt/data/documents/
documents
        Name:                   documents
        UUID:                   82224209-acdb-fc46-b2af-daeff385d841
        Parent UUID:            -
        Received UUID:          -
        Creation time:          2020-11-25 18:37:03 +0000
        Subvolume ID:           257
        Generation:             14
        Gen at creation:        7
        Parent ID:              5
        Top level ID:           5
        Flags:                  -
        Snapshot(s):
                                documents/snapshots
```

When the command is run against the snapshot subvolume, the `Parent ID: 257` does indicate that this subvolume is a child of the source subvolume.

```
deep@ubuntu-vm:/mnt/data/documents$ sudo btrfs subvolume show /mnt/data/documents/snapshots
documents/snapshots
        Name:                   snapshots
        UUID:                   febf959f-7bfd-ae41-81e8-c73707ec07d6
        Parent UUID:            82224209-acdb-fc46-b2af-daeff385d841
        Received UUID:          -
        Creation time:          2020-11-25 19:43:36 +0000
        Subvolume ID:           258
        Generation:             13
        Gen at creation:        13
        Parent ID:              257
        Top level ID:           257
        Flags:                  -
        Snapshot(s):
```

As a belt-and-suspenders check, let's run the `btrfs subvolume list` command against the source and snapshot subvolumes to determine the same information.

```
deep@ubuntu-vm:/mnt/data/documents$ sudo btrfs subvolume list /mnt/data/documents
ID 257 gen 14 top level 5 path documents
ID 258 gen 13 top level 257 path snapshots

deep@ubuntu-vm:/mnt/data/documents$ sudo btrfs subvolume list /mnt/data/documents/snapshots
ID 257 gen 14 top level 5 path documents
ID 258 gen 13 top level 257 path documents/snapshots
```

### Recovering a Snapshot
Now that we have created a snapshot, what can we do with it?  The snapshot is just another subvolume that contains the data of the source subvolume.  Both appear as directories.  Therefore, you could just copy files from the snapshot into the source subvolume as a primitive way of restoring.  That's just like backing up to a different location using `cp` or `rsync` and recovering the copies of the desired files and directories.  Where's the fun in that?  

The `btrfs subvolume set-default` command is a more eloquent way of shifting a mount-point to a different subvolume. 

Prior to proceeding with any type or restoration, let's destroy some data in the `/mnt/data/documents` subvolume by deleting the `files` subdirectory.  This will help prove whether or not the restoration was successful.

```
deep@ubuntu-vm:/mnt/data/documents$ sudo rm -rf files

deep@ubuntu-vm:/mnt/data/documents$ ls *
notes:
note1.txt  note2.txt  note3.txt

snapshots:
files  notes
```

Let's also take a moment to assess the existing subvolumes.  The snapshots subvolume is the one which we are interested in at the moment.  This subvolume has a Subvolume Id of 258.  

Subvolume Id | Parent Subvolume Id | Name | Path
-------------|---------------------|------|-----
5 | 0 | <FS_TREE> | /mnt/data
257 | 5 | documents | /mnt/data/documents
258 | 257 | snapshots | /mnt/data/documents/snapshots
<br/>

The current default subvolume is 5 and we are going to change it to 258.  

```
deep@ubuntu-vm:/mnt/data/documents$ sudo btrfs subvolume set-default 258 /mnt/data
```

Next, we need to unmount `/dev/sdb`.

```
deep@ubuntu-vm:/mnt/data/documents$ sudo umount -l /dev/sdb
```

Now, let's mount `/dev/sdb` to the `/mnt/data` location.

```
deep@ubuntu-vm:/mnt$ sudo mount /dev/sdb /mnt/data
```

Finally, let's navigate to the `/mnt/data` directory and recursively list the contents to verify that the restore was successful.

```
deep@ubuntu-vm:/mnt$ cd /mnt/data

deep@ubuntu-vm:/mnt/data$ ls -R *
files:
file1.txt  file2.txt  file3.txt

notes:
note1.txt  note2.txt  note3.txt
```

To go back to the way things were prior to changing the default subvolume. The following commands can be performed,

```
deep@ubuntu-vm:/mnt/data$ sudo btrfs subvolume set-default 5 /mnt/data

deep@ubuntu-vm:/mnt/data$ sudo umount -l /dev/sdb

deep@ubuntu-vm:/mnt/data$ sudo mount /dev/sdb /mnt/data

deep@ubuntu-vm:/mnt/data$ cd documents

deep@ubuntu-vm:/mnt/data/documents$ ls -R *
notes:
note1.txt  note2.txt  note3.txt

snapshots:
files  notes

snapshots/files:
file1.txt  file2.txt  file3.txt

snapshots/notes:
note1.txt  note2.txt  note3.txt

snapshots-ro:
files  notes  snapshots

snapshots-ro/files:
file1.txt  file2.txt  file3.txt

snapshots-ro/notes:
note1.txt  note2.txt  note3.txt

snapshots-ro/snapshots:
```

### Creating a Read-Only Snapshot for Transmission
Btrfs allows for the transmission of a snapshot to any other device which also contains storage using the Btrfs filesystem.  In order for this to function properly, the snapshot must be creating in read-only mode.

#### Creating the Read-Only Snapshot
Let's go ahead and create a read-only snapshot of the subvolume `/mnt/data/documents` into the snapshot location `/mnt/data/documents/snapshots-ro`.  Since we want to make a read-only snapshot, we  must provide the `-r` parameter to the `btrfs subvolume snapshots` command.  

```
deep@ubuntu-vm:/mnt/data/documents$ sudo btrfs subvolume snapshot -r /mnt/data/documents /mnt/data/documents/snapshots-ro
Create a readonly snapshot of '/mnt/data/documents' in '/mnt/data/documents/snapshots-ro'
```

### Transmitting the Snapshot
The `btrfs send` and `btrfs receive` commands can be used in conjunction to send a snapshot from a host system and receive the snapshot by a target system.  This snapshot can then be mounted and fully functional on the target system.

In our example, we will send the read-only snapshot created in the previous step to a newly mounted btrfs filesystem at `/mnt/restore`.

```
deep@ubuntu-vm:~$ sudo btrfs send /mnt/data/documents/snapshots-ro/ | sudo btrfs receive /mnt/restore/
```

## Summary
Btrfs is a powerful and streamlined implementation within Linux.  
___
## Additional Commands
Although not covered in this tutorial, here are some additional commands that can be useful when using the Btrfs file system.

### Device Usage

```
deep@ubuntu-vm:~$ sudo btrfs device usage /mnt/data
/dev/sdb, ID: 1
   Device size:            10.00GiB
   Device slack:              0.00B
   Data,single:             8.00MiB
   Metadata,DUP:          512.00MiB
   System,DUP:             16.00MiB
   Unallocated:             9.48GiB
```

### Device Statistics

```
deep@ubuntu-vm:~$ sudo btrfs device stats /mnt/data
[/dev/sdb].write_io_errs    0
[/dev/sdb].read_io_errs     0
[/dev/sdb].flush_io_errs    0
[/dev/sdb].corruption_errs  0
[/dev/sdb].generation_errs  0
```

### Filesystem Show

```
deep@ubuntu-vm:~$ sudo btrfs filesystem show /mnt/data
Label: none  uuid: c660a282-d8b8-4273-a7eb-f16f648aac9b
        Total devices 1 FS bytes used 208.00KiB
        devid    1 size 10.00GiB used 536.00MiB path /dev/sdb
```

### Filesystem Usage

```
deep@ubuntu-vm:~$ sudo btrfs filesystem usage /mnt/data
Overall:
    Device size:                  10.00GiB
    Device allocated:            536.00MiB
    Device unallocated:            9.48GiB
    Device missing:                  0.00B
    Used:                        352.00KiB
    Free (estimated):              9.48GiB      (min: 4.75GiB)
    Data ratio:                       1.00
    Metadata ratio:                   2.00
    Global reserve:                3.25MiB      (used: 0.00B)

Data,single: Size:8.00MiB, Used:64.00KiB (0.78%)
   /dev/sdb        8.00MiB

Metadata,DUP: Size:256.00MiB, Used:128.00KiB (0.05%)
   /dev/sdb      512.00MiB

System,DUP: Size:8.00MiB, Used:16.00KiB (0.20%)
   /dev/sdb       16.00MiB

Unallocated:
   /dev/sdb        9.48GiB
```  

### Filesystem Disk Free

```
deep@ubuntu-vm:~$ sudo btrfs filesystem df /mnt/data
Data, single: total=8.00MiB, used=64.00KiB
System, DUP: total=8.00MiB, used=16.00KiB
Metadata, DUP: total=256.00MiB, used=128.00KiB
GlobalReserve, single: total=3.25MiB, used=0.00B
```

### Filesystem Disk Usage

```
deep@ubuntu-vm:~$ sudo btrfs filesystem du /mnt/data
     Total   Exclusive  Set shared  Filename
     0.00B       0.00B           -  /mnt/data/documents
     0.00B       0.00B       0.00B  /mnt/data
```