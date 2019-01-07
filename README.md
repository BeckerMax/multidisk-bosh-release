# How it should work:

- Add section to manifest to include an additional persistent disk via links.
- Add release to manifest which includes information about partitioning and to which path to mount

- DEPLOY:
  - Disk gets created by BOSH via CPI and attached to instances 
  - If disk is not yet partitioned release will partiton it.
  - Mount disk to the given location

- RECREATE:
  - Release unmounts the disk
  - BOSH detaches the disk from VM
  - BOSH creates new VM
  - BOSH attaches the Disk
  - Release mounts the disk

# More Context:

- https://www.pivotaltracker.com/n/projects/1456570/stories/141576895
- https://starkandwayne.com/blog/bosh-multiple-disks/

# Investigations:

Can I attach a normal disk(persisitent_disk) and multiple other disks(persisten_disks)
 -> no in a multidisk scenarion you (or the release) always needs to take care themselves of all disk partitioning mounting
What happens in recreation with multiple disks?
  - case: not mounted => detaches and attaches
  - case: mounted => Recreation hangs in detaching disk with:
    - Detaching disk vol-0846db733d9cade05
    - D, [2019-01-07T12:26:46.046547 #14224] [canary_update(zookeeper/b6c33f33-d14b-4309-ac07-e7f51ed770b0 (0))] DEBUG -- DirectorJobRunner: [external-cpi] [cpi-385036] request: {"method":"detach_disk","arguments":["i-07dea98148649c754","vol-0846db733d9cade05"],"context":{"director_uuid":"4e5d8fd4-d402-40f5-9aeb-759605d35abe","request_id":"cpi-385036"},"api_version":2} with command: /var/vcap/jobs/aws_cpi/bin/cpi
  - case I manually unmount with `umount /var/vcap/firstDisk`
    - recreate triggers successful detached and attached
    -  After mounting data is accessible again
Can I create snapshots?
  - `take-snapshot` does create a snapshot of both disks
What happens in case of resize?
  - Disk gets attached again but is neither partitioned nor mounted
  - after partitioning and mounting the old data is now lost -> was there no
    migration or did I delete it with repartitioning

- Unmounten der Disk im Release im drain script 
  - Passiert dann also vor monit stop, also könnte es sein, dass noch Daten geschrieben werden. 
  - Das kann eine Limitaiton sein. Würde besser werden mit Post_stop. 
- Wieviel kann ich wieder verwenden vom Agent Code? Mounten?.. 

Random questions:
- Does disk resizing work with 2 disks?


# General Notes:

How to partiton and mount a disk with parted:

- Find the attached disk
  - lsblk
  
- Create partition table (gpt):
  - parted /dev/vdc
  - (parted) mklabel gpt
  - (parted) print (see Disk size)
  - (parted) mkpart
    - Partition Name: vdc1
    - File system type: ext4
    - Start? 0
    - End? 5369

- Create ext4 filesystem
  - mkfs.ext4 -L secondDisk /dev/vdc1


- Mounting to /var/vcap/second_disk
  - mkdir /var/vcap/second_disk
  - mount -t ext4 /dev/vdc1 /var/vcap/second_disk

- Unmounting before detaching:
  - umount /var/vcap/firstDisk

## OUTPUT

### BEFORE:
```
dummy/740655c1-2693-496b-8a8d-e2bac23e088a:/dev# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0    1G  0 disk
└─vdb1 252:17   0 1022M  0 part /var/vcap/store
vdc    252:32   0    1G  0 disk
vda    252:0    0   25G  0 disk
├─vda2 252:2    0    2G  0 part [SWAP]
├─vda3 252:3    0 20.2G  0 part /var/vcap/data
└─vda1 252:1    0  2.9G  0 part /
```

```
dummy/740655c1-2693-496b-8a8d-e2bac23e088a:/dev# parted -l
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 1074MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name              Flags
 1      1049kB  1073MB  1072MB  ext4         bosh-partition-0


Error: /dev/vdc: unrecognised disk label
Model: Virtio Block Device (virtblk)
Disk /dev/vdc: 1074MB
Sector size (logical/physical): 512B/512B
Partition Table: unknown
Disk Flags:

Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system     Flags
 1      32.3kB  3071MB  3071MB  primary  ext4
 2      3071MB  5161MB  2090MB  primary  linux-swap(v1)
 3      5162MB  26.8GB  21.7GB  primary  ext4

```



### AFTER:

```
dummy/740655c1-2693-496b-8a8d-e2bac23e088a:/dev# lsblk
NAME   MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
vdb    252:16   0    1G  0 disk
└─vdb1 252:17   0 1022M  0 part /var/vcap/store
vdc    252:32   0    1G  0 disk
└─vdc1 252:33   0 1024M  0 part
vda    252:0    0   25G  0 disk
├─vda2 252:2    0    2G  0 part [SWAP]
├─vda3 252:3    0 20.2G  0 part /var/vcap/data
└─vda1 252:1    0  2.9G  0 part /
```


```
dummy/740655c1-2693-496b-8a8d-e2bac23e088a:~# parted -l
Model: Virtio Block Device (virtblk)
Disk /dev/vdb: 1074MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name              Flags
 1      1049kB  1073MB  1072MB  ext4         bosh-partition-0


Model: Virtio Block Device (virtblk)
Disk /dev/vdc: 1074MB
Sector size (logical/physical): 512B/512B
Partition Table: gpt
Disk Flags:

Number  Start   End     Size    File system  Name     Flags
 1      17.4kB  1074MB  1074MB  ext4         primary


Model: Virtio Block Device (virtblk)
Disk /dev/vda: 26.8GB
Sector size (logical/physical): 512B/512B
Partition Table: msdos
Disk Flags:

Number  Start   End     Size    Type     File system     Flags
 1      32.3kB  3071MB  3071MB  primary  ext4
 2      3071MB  5161MB  2090MB  primary  linux-swap(v1)
 3      5162MB  26.8GB  21.7GB  primary  ext4

```

