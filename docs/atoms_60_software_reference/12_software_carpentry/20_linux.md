# Linux {#linux}

Assigned: Andrea

## Background reading

- UNIX
- Linux
- free software; open source software.

<!--

### Can you do the following?

- Log in the Duckiebot from the laptop;
- Log in the Duckiebot from the laptop, without using password;


### Can you do the following?

- Copy a file from the Duckiebot to the laptop;
- Copy a file from the laptop to the Duckiebot; -->


## Ubuntu packaging

### `apt install`

TODO: to write

### `apt update`

TODO: to write

### `apt dist-upgrade`

TODO: hold back packages

## Measuring resource usage


### `pgrep`

TODO: to write

### Testing SD Card and disk speed

Test SD Card (or any disk) speed using the following commands,
which write to a file called `![filename]`.

    $ dd if=/dev/zero of=![filename] bs=500K count=1024
    $ sync
    $ echo 3 | sudo tee /proc/sys/vm/drop_caches
    $ dd if=![filename] of=/dev/null bs=500K count=1024
    $ rm ![filename]

Note the `sync` and the `echo` command are very important.

Example results:

    524288000 bytes (524 MB, 500 MiB) copied, 30.2087 s, 17.4 MB/s
    524288000 bytes (524 MB, 500 MiB) copied, 23.3568 s, 22.4 MB/s

That is write 17.4 MB/s, read 22 MB/s.

### Measuring CPU usage using `htop`

You can use `htop` to monitor CPU usage.

    $ sudo apt install htop

### Measuring I/O usage using `iotop`

    $ sudo apt install iotop

## How to burn an image to an SD card {#howto-burn-image}

<div class='requirements' markdown='1'>

Requires:

- A blank SD card.
- An image file to burn.
- An Ubuntu computer with an SD reader.

Results:

- A burned image.

</div>

### Finding your device name for the SD card

First, find out what is the device name for the SD card.

Insert the SD Card in the slot.

Run the command:

    $ sudo fdisk -l

Find your device name, by looking at the sizes.

For example, the output might contain:

    Disk /dev/mmcblk0: 14.9 GiB, 15931539456 bytes, 31116288 sectors
    Units: sectors of 1 * 512 = 512 bytes
    Sector size (logical/physical): 512 bytes / 512 bytes
    I/O size (minimum/optimal): 512 bytes / 512 bytes

In this case, the device is `/dev/mmcblk0`. That will be the `![device]`
in the next commands.

You may see `/dev/mmcblk0pX` or a couple of similar entries for each partition on the card,
where `X` is the partition number. If you don't see anything like that, take out
the SD card and run the command again and see what disappeared.

### Unmount partitions

Before proceeding, unmount all partitions.

Run `df -h`. If there are partitions like `/dev/mmcblk0p![n]`, then unmount
each of them. For example:

    laptop $ sudo umount /dev/mmcblk0p1
    laptop $ sudo umount /dev/mmcblk0p2


### Burn the image

Now that you know that the device is `![device]`,
you can burn the image to disk.

Let the image file be `![image file]`.

Burn the image using the command `dd`:

    laptop $ sudo dd of=![device] if=![image file] status=progress bs=4M

Note: Use the name of the device, without partitions. i.e., `/dev/mmcblk0`, not
`/dev/mmcblk0pX`.

## How to shrink an image {#howto-shrink-image}

<div class='requirements' markdown='1'>

Requires:

- An image file to burn.
- An Ubuntu computer.

Results:

- A shrunk image.

</div>

Note: Majority of content taken from [here](http://www.aoakley.com/articles/2015-10-09-resizing-sd-images.php)

We are going to use the tool `gparted` so make sure it's installed

    laptop $ sudo apt-get install gparted
    
Let the image file be `![image file]`.
Run the command:

    laptop $ sudo fdisk -l ![image file]
should give you something like:
```
Device                       Boot  Start      End  Sectors  Size Id Type
duckiebot-RPI3-LP-aug15.img1        2048   131071   129024   63M  c W95 FAT32 (LBA)
duckiebot-RPI3-LP-aug15.img2      131072 21219327 21088256 10.1G 83 Linux
```
Take note of the start of the Linux partition (in our case 131072), let's call it `![start]`
Now we are going to mount the Linux partition from the image:

    laptop $ sudo losetup /dev/loop0 imagename.img -o $((![start]*512))

and then run `gparted`:

    laptop $ sudo gparted /dev/loop0

In `gparted` click on the partition and click "Resize" under the "Partition" menu. Resize drag the arrow or enter a size
that is equal to the minimum size plus 20MB

Note: This didn't work well for me - I had to add much more than 20MB for it to work. 

Click the "Apply" check mark. *Before* closing the final screen click through the arrows in the dialogue box 
to find a line such a *"resize2fs -p /dev/loop0 1410048K"*. Take note of the new size of your partition. Let's
call it `![new size]`.

Now remove the loopback on the 2nd partition and setup a loopback on the whole image and run `fdisk`:

    laptop $ sudo losetup -d /dev/loop0
    laptop $ sudo losetup /dev/loop0 ![image file]
    laptop $ sudo fdisk /dev/loop0
    
    Command (m for help): *d*
    Partition number (1,2, default 2): *2*
    Command (m for help): *n*
    Partition type
    p   primary (1 primary, 0 extended, 3 free)
    e   extended (container for logical partitions)
    Select (default p): *p*
    Partition number (2-4, default 2): *2*
    First sector (131072-62521343, default 131072): *![start]*
    Last sector, +sectors or +size{K,M,G,T,P} (131072-62521343, default 62521343): *+![new size]*

(Note: on the last line to include the `+` and the `K` as part of the size.)

    Created a new partition 2 of type 'Linux' and of size 10.1 GiB.

    Command (m for help): *w*
    The partition table has been altered.
    Calling ioctl() to re-read partition table.
    Re-reading the partition table failed.: Invalid argument

    The kernel still uses the old table. The new table will be used at the next reboot or after you run partprobe(8) or kpartx(8).

Disregard the final error.

You partition has now been resized and the partition table has been updated. Now we will remove the loopback and then 
truncate the end of the image file:

    laptop $ fdisk -l /dev/loop0

```
Device       Boot  Start      End  Sectors  Size Id Type
/dev/loop0p1        2048   131071   129024   63M  c W95 FAT32 (LBA)
/dev/loop0p2      131072 21219327 21088256 10.1G 83 Linux
```
Note down the end of the second partition (in thise case 21219327). Call this `![end]`.

    laptop $ sudo losetup -d /dev/loop0
    laptop $ sudo truncate -s $(((![end]+1)*512)) ![image file]

You now have a shrunken image file. A further idea is to compress it:

    laptop $ xz ![image file]
    


