---
title: "Encrypting additional drives with LUKS on Linux"
date: "2020-06-29"
categories: 
  - "devops-sysadmin"
  - "development"
tags: 
  - "encryption"
  - "luks"
  - "security"
  - "sysadmin"
  - "ubuntu"
coverImage: "encryption-encoding-hashing.jpg"
---

Encryption is king nowadays with everyone having mobile devices. We have a significant number of people on laptops that travel around and also workstations that live in open plan offices. This means we encrypt all of our disks... just in case. 99% of the time is super simple to do as most OS installers give you the option to do it, some now ven enforce it as a default option. This post however is about adding an additional disk to the system and making it automatically mount on system startup.

So let me set the scene, we have a data-scientist that's running out of disk space for a task they are running on their Ubuntu 18.04 Workstation. At some point the workstation had an upgrade to the HDD in the past to a shiny new SSD, and the old 4Tb spinning disk was left in the chassis that they want to use for this very specific task.

Now this workstation has been through a couple of data-scientists over the last 12 months and unfortunately the LUKS password that had been set up for the old spinning disk has gone walkabouts... so the plan is as follows

1. flatten the old disk and set up a new partition using the whole disk
2. generate a new secure encryption key
3. set up LUKS encryption on the new partition
4. use Ext4 as a filesystem
5. enable auto decryption of the disk
6. add the new partition to the fstab to mount on system startup

N.B Assume that we are running everything as the root user

## Flatten the disk

As we cant recover anything we are going to flatten the disk using `parted` (`apt install parted` to install) to allow is to create a partition greater than 2Tb, but first we are going to identify the disk we are working with... I tend to favour using either `fdisk -l` or as a more concise option `lsblk -p` which gives us a an easy to interpret overview something like:

```
/dev/sda                              8:0    0   1.8T  0 disk  
├─/dev/sda1                           8:1    0   512M  0 part  /boot/efi
├─/dev/sda2                           8:2    0   732M  0 part  /boot
└─/dev/sda3                           8:3    0   1.8T  0 part  
  └─/dev/mapper/sda3_crypt          253:0    0   1.8T  0 crypt 
    ├─/dev/mapper/ubuntu--vg-root   253:1    0   1.8T  0 lvm   /
    └─/dev/mapper/ubuntu--vg-swap_1 253:2    0   976M  0 lvm   [SWAP]
/dev/sdb                              8:16   0   3.7T  0 disk  
```

I can tell from this that we are looking at using the disk that is currently at /dev/sdb and its showing as being 3.7Tb in size.

Great... now to set up our new partition using the command `parted /dev/sdb` which gives us an interactive shell to work with (you can see the prompts in the output below are prefixed with (parted)

```
GNU Parted 3.2                                                                    
Using /dev/sdb                                                                           
Welcome to GNU Parted! Type 'help' to view a list of commands.                               
(parted) mklabel gpt                                                                     
```

The command `mklabel gpt` will wipe the partition table for `/dev/sdb` and give us a clean slate to work from

```
(parted) unit TB 
```

We now set parted to think in Terabytes as the default reference size using the command above.

```
(parted) mkpart primary 0.00TB 3.70TB                            
```

Now we get to create the actual partition. You can see from the command above that we are using the command `mkpart` and telling it to create a `primary` partition type.

```
(parted) print                                                                                            
Model: ATA WDC WD4005FZBX-0 (scsi)                                                                                                         
Disk /dev/sdb: 4.00TB                                        
Sector size (logical/physical): 512B/4096B                               
Partition Table: gpt                                                                    
Disk Flags:                                                                        
                                                                                         
Number  Start   End     Size    File system  Name     Flags                
 1      0.00TB  4.00TB  4.00TB               primary                                    
                                                                                                       
```

We can chek everything went smoothly using the `print` command which gives us confirmation that a new primary partition is present. We can now leave `parted` with a simple.

```
(parted) quit  
```

And we can now use `fdisk -l` or `lsblk -p` to see that we now have a partition waiting for us at `/dev/sdb1`.

```
/dev/sda                              8:0    0   1.8T  0 disk  
├─/dev/sda1                           8:1    0   512M  0 part  /boot/efi
├─/dev/sda2                           8:2    0   732M  0 part  /boot
└─/dev/sda3                           8:3    0   1.8T  0 part  
  └─/dev/mapper/sda3_crypt          253:0    0   1.8T  0 crypt 
    ├─/dev/mapper/ubuntu--vg-root   253:1    0   1.8T  0 lvm   /
    └─/dev/mapper/ubuntu--vg-swap_1 253:2    0   976M  0 lvm   [SWAP]
/dev/sdb                              8:16   0   3.7T  0 disk  
└─/dev/sdb1                           8:17   0   3.7T  0 part 
```

## Generating an encryption key

Our disk is now ready for use, but not yet encrypted, so our next step is to create a key that can be used when we encrypt the disk. As we are going to be mounting it automatically we want to use a keyfile to store the key. You can of course create a key by mashing the keys on the keyboard, but I tend to prefer letting something else do the hard part for me.

![](images/encryption-encoding-hashing-1.jpg)

First we create somewhere to store the key... I opted for,

```
mkdir -p /etc/crypt/keys
```

But feel free to put it wherever you want just as long as its only accessible by the `root` user. Next we generate the keyfile using the command:

```
dd bs=512 count=4 if=/dev/urandom of=/etc/crypt/keys/sdb1 iflag=fullblock
```

Here I am using `/dev/urandom` as my randomness generator, but you could use any valid generator of your choice. With this set of parameted `dd` will read the stream of "randomeness" and write 2048 bytes to our keyfile at `/etc/crypt/keys/sdb1`. If you want to be a little more complex about teh size and shape of your key then have a look at [https://man7.org/linux/man-pages/man1/dd.1.html](https://man7.org/linux/man-pages/man1/dd.1.html)

## Encrypting the Disk

![](images/luks-logo-cropped.png)

Hopefully it will already be installed because you encrypted your root disk at installation, but if not you can run `apt install cryptsetup` to get going.

The command to do the encryption is actually very simple.

```
cryptsetup luksFormat /dev/sdb1 /etc/crypt/keys/sdb1
```

You can see that using the `cryptsetup` tool we are asking it to execute teh command `luksFormat` but while it says format in the command this is a little misleading as it doesn't actually format the disk but just rewrites a portion of bytes at the beginning of the partition to enable encryption. we then tell it the partition we want encrypting, here its `/dev/sdb1` and finally we pass in the keyfile we just generated and saved at `/etc/crypt/keys/sdb1`. If you omit the keyfile it will still encrypt teh disk but will prompt you to enter the key manually.

As soon as you press enter you will be warned of teh danager of what you are doing... so double check you are encrypting the right partition and follow the instructions that should look something like :

```
WARNING!                                                                                   
========                                                                                                         
This will overwrite data on /dev/sdb1 irrevocably.                                         
                                                                                       
Are you sure? (Type uppercase yes): YES                                                            
Command successful.                  
```

And that's it... the disk is encrypted and ready to use. There are a few ways you can now work with the disk. the quickest and easiest is to just decrypt the disk manually using cryptsetup to `open` the disk.

```
cryptsetup open /dev/sdb1 sdb1_crypt -d /etc/crypt/keys/sdb1
```

Here we open `/dev/sdb1` and give is a new name of sdb1\_crypt and we unlock it using the `-d` argument to tell it the keyfile we generated before.

That is the dis decrypted and ready to roll... you can now use `fdisk -l` or `lsblk -p` to confirm that it is now available at `/dev/mapper/sdb1_crypt`.

```
/dev/sda                              8:0    0   1.8T  0 disk  
├─/dev/sda1                           8:1    0   512M  0 part  /boot/efi
├─/dev/sda2                           8:2    0   732M  0 part  /boot
└─/dev/sda3                           8:3    0   1.8T  0 part  
  └─/dev/mapper/sda3_crypt          253:0    0   1.8T  0 crypt 
    ├─/dev/mapper/ubuntu--vg-root   253:1    0   1.8T  0 lvm   /
    └─/dev/mapper/ubuntu--vg-swap_1 253:2    0   976M  0 lvm   [SWAP]
/dev/sdb                              8:16   0   3.7T  0 disk  
└─/dev/sdb1                           8:17   0   3.7T  0 part  
  └─/dev/mapper/sdb1_crypt          253:3    0   3.7T  0 crypt /mnt/4tb-1
```

This tells us that the newly decrypted disk is now available at `/dev/mapper/sdb1_crypt` and is a volume of 3.7Tb... Exactly what we were hoping for!

All finished with your encrypted disk... you can just as easily close it again using:

```
cryptsetup close sdb1_crypt
```

## Setting up the Filesystem

Ok, we have an encrypted partition, we can decrypt it but we cant mount it yet as we don't have a file system to work with. Let's take care of that real quick by opening up the partition again.

```
cryptsetup open /dev/sdb1 sdb1_crypt -d /etc/crypt/keys/sdb1
```

And now that its available we are going to set up an ext4 filesystem using the command `mkfs.ext4 /dev/mapper/sdb1_crypt` which, all going according to plan, should look something like:

```
mke2fs 1.44.1 (24-Mar-2018)                                                   
Creating filesystem with 976753664 4k blocks and 244195328 inodes               
Filesystem UUID: d797be67-c53e-49d3-897e-c624b21a22d3                                      
Superblock backups stored on blocks:                                                       
        32768, 98304, 163840, 229376, 294912, 819200, 884736, 1605632, 2654208,        
        4096000, 7962624, 11239424, 20480000, 23887872, 71663616, 78675968,                
        102400000, 214990848, 512000000, 550731776, 644972544                          
                                                                                                       
Allocating group tables: done                                                            
Writing inode tables: done                                                     
Creating journal (262144 blocks): done                                                        
Writing superblocks and filesystem accounting information: done
```

we are now good to go... lets try mounting the filesystem with

```
mount -t ext4 /dev/mapper/sdb1_crypt /mnt
```

I'm just mounting straight to `/mnt` but obviously this can be any folder you want. If the command worked we can easily confirm it with a quick `df` -h:

```
Filesystem                   1K-blocks      Used  Available Use% Mounted on
udev                          32846708         0   32846708   0% /dev
tmpfs                          6578140      2308    6575832   1% /run
/dev/mapper/ubuntu--vg-root 1919562064 993193376  828790500  55% /
tmpfs                         32890688       200   32890488   1% /dev/shm
tmpfs                             5120         4       5116   1% /run/lock
tmpfs                         32890688         0   32890688   0% /sys/fs/cgroup
/dev/sda2                       721392    276068     392860  42% /boot
/dev/sda1                       523248      6232     517016   2% /boot/efi
/dev/mapper/sdb1_crypt      3844637680   0 3844637680   1% /mnt
```

Excellent... you can now start working with your new partition... however lets un-mount and close the drive quickly with a

```
umount /mnt && cryptsetup close sdb1_crypt
```

And then we can move onto...

## Automatic Decryption

This is a lot simpler that you may realise... all we need to do is add a new line to the file `/etc/crypttab`! But first we need one last piece of information we don't yet have, but we can easily get with the command

```
sudo cryptsetup luksDump /dev/sdb1 | grep "UUID"
```

This will use `luksDump` to get information about the encrypted partition and then uses `grep` to specifically target the property UUID which we will need to identify the partition in the next step.

Now in your favourite editor of choice add the following line, replacing the spoof UUID here with the one we just found.

```
sdb1_crypt UUID=1111111111-2222-3333-4444-555555555555 /etc/crypt/keys/sdb1 luks
```

Here we are giving the decrypted volume a unique label for teh decrypted label to be made available at the appropriate `/dev/mapper/*` location. We also specify the UUID to identify the partition to decrypt... we could use the path `/dev/sdb1` but using the UUID is more explicit and prevents any confusion if another partition happens to present itself as `/dev/sdb1` at some point in the future. Third we have the path to our newly generated keyfile and finally we have the encryption mode that we are using for encryption which here is `luks`. For more info on crypttab have a look at [https://www.freedesktop.org/software/systemd/man/crypttab.html](https://www.freedesktop.org/software/systemd/man/crypttab.html)

We can now test that auto decryption is working using:

```
cryptdisks_start sdb1_crypt
```

which if successful should have an output like:

```
 * Starting crypto disk...                                                                                                                                                 
 * sdb1_crypt: INSECURE MODE FOR /etc/crypt/keys/sdb1, see /usr/share/doc/cryptsetup/README.Debian.                                                                        
 * sdb1_crypt (starting)..                                                                                                                                                
 * sdb1_crypt (started)...  
```

All that's left to do now is set up

## Auto-mount the filesystem

Hopefully we now are on really familiar ground... we can now treat `/dev/mapper/sdb1_crypt` as a bog standard ext4 partition that can be mounted via the `/etc/fstab` by adding the line:

```
/dev/mapper/sdb1_crypt /mnt ext4    defaults   0       2
```

As you can see its pretty ordinary, exactly as you would expect, obviously swapping out `/mnt` with the location of your choice to mount the filesystem. If you are not wholly familiar with `fstab` then its definitely worth having a look at [https://help.ubuntu.com/community/Fstab](https://help.ubuntu.com/community/Fstab) as it gives a good overview for those who are new to it...

Finally we can check that it all works with:

```
mount -a
```

And at this point I am pretty sure I can hear the fat lady singing...
