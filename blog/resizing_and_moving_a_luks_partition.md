## The situation

So this is a bit of a funny one - I had Windows and mint co-existing on my laptop, but then decided to make use of a spare drive I had and move windows to live on an external drive. This went absolutely fine, so I went about making use of the leftover space for my mint install. Because I had Windows installed, my 256GB SSD was laid out something like this;

* sda1 - 100MB Windows System Reserved (contains the boot loader)
* sda2 - 80GB Windows 8
* sda3 - extended partition
* 	sda5 - 300MB unencrypted /boot
* 	sda6 - 160GB luks encrypted partition (mounts as /)
* 	sda7 - 8GB luks encrypted swap

## My goal

I wanted to do two things, ideally - firstly, I wanted to get that extra 80GB of space being used by Windows into my root file system (buried in sda6's luks volume). Secondly (and this was only a nice-to-have), I wanted to get everything out of the extended partition - because having an extended partition than spans an entire disk, especially when you've only got a linux system on it, seems pointless.

## The adventure begins...

Before I got started, I compiled a couple of things I thought I'd need;

* an external hard drive (I used a 1TB drive, so that I had space for multiple copies of the data at different points in time)
* a live CD/USB (I used a live mint 16 installer flashed to a USB drive, because I had one handy)

I got rid of the first two partitions, no problem - this left me with about 80GB of usable space at the beginning of the drive. I then extended the extended partition (excuse the phrasing) to cover the whole drive, and moved sda5 (boot) to the beginning of this space. That left me with something like the following;

* sda3 - extended partition
* 	sda5 - 300MB unencrypted /boot
* 	80GB unallocated space
* 	sda6 - 150GB luks encrypted partition
* 	sda7 - 8GB luks encrypted swap

## The problems

As the title suggests, this is where the problems arose.

At first I tried to go about the shuffling of space into sda6 - I thought I'd resize the partition, and then luksOpen it and resize2fs the mapper device...turns out libparted (and hence Gparted) don't support luks. And, as it turns out, neither do most other tools.

So I looked around, and some people thought you could do this by fiddling with the partition table by hand, calculating the offsets, etc - but that's not for me. And a lot of the people broke their system in the process - so it probably wasn't for them either.

## My solution

# Take a backup

I started by taking a copy of the entire disk as it stood - running something like the below will do the trick for that. Now, I have a copy to revert to if I screw this all up.

`dd if=/dev/sda of=/media/somedisk/sda.img`

# Prepare for the move

I started of by taking a copy of my boot partition;

`dd if=/dev/sda5 of=/media/somedisk/sda5.img`

Then I luksOpen-ed sda6 (`cryptsetup luksOpen sda6 sda6_crypt`) and dd-ed that to a file alongside the other, like so;

`dd if=/dev/mapper/sda6_crypt of=/media/somedisk/sda6_crypt.img`

the important bit about this is it's the volume inside the partition, not the partition itself - that way I can create a new luks partition of the size I want later on, restore this volume into that partition, and resize the filesystem to fill the partition.

# Prepare the disk and partitions for restoring

Next comes the scary bit - now that we've taken a copy of everything we want (I've ignored sda7 as it's just swap and can be rereated), we create a new partition table and start again. I did the easy one first - I created a 300MB active partition at the beginning of the disk to be my new /boot - this was sda1. Then I created an unformatted primary partition, 230GB big to house my new luks crypt-ed root file system. This became sda2. I made the remaining space into another unformatted primary partition, to be my new luks crypt-ed swap partition, which was sda3.

Then I used luksFormat to make the two new partitions into crypt partitions, like so;

`cryptsetup --verify-passphrase luksFormat /dev/sda2`

`cryptsetup --verify-passphrase luksFormat /dev/sda3`

# Put the file system back

I then mounted sda2 (the new crypt partition to home my root file system) using luksOpen in the same manner as earlier, and set about dd-ing the file of my old sda6_crypt onto the new mapper device;

`dd if=/media/somedisk/sda6_crypt.img of=/dev/mapper/sda2_crypt`

because you're writing to the mapper device, and not a file, it's not as obvious what's going on while you're runing that command - running a `kill -USR1 $PID` with the process ID of the dd command from another terminal will spit out a status on the terminal where dd's running.	

# Resize the file system

This bit's really easy - with the mapper device still mounted, just run

`resize2fs /dev/mapper/sda2_crypt`

# Recreate the swap space

With the mapper device for swap still mounted (sda3 in my case), just run

`mkswap /dev/mapper/sda3_crypt`

# Update the boot loader

I started by putting my old boot partition back, so I had something to work with - that was done like so;

`dd if=/media/somedisk/sda5.img of=/dev/sda1`

next I needed to set grub up again, so I mounted the mapper device as below;

`mount /dev/mapper/sda2_crypt /mnt`

`mount --bind /dev /mnt/dev`

`mount --bind /dev/pts /mnt/dev/pts`

`mount --bind /proc /mnt/proc`

`mount --bind /sys /mnt/sys`

Then chroot-ed into the newly mounted file system;

`chroot /mnt`

mounted the boot partition

`mount /dev/sda1 /boot`

tweaked my fstab and crypttab files to match the new partition layout, and ran the following;

`update-initramfs -u`

`update-grub`

and unmounted everything again

`umount /boot`

`exit`

`umount /mnt/sys`

`umount /mnt/proc`

`umount /mnt/dev/pts`

`umount /mnt/dev`

`umount /mnt`

## Conclusion

I've achieved what I needed to, which was to make use of all of the space on my drive for my crypt partition, and move them out of the extended partition they lived in. I didn't quite do it in the way I thought I would be - I've actually copied everything off the drive, flattened it, and recreated it with a layout I'm happy with.

For future reference I'd always try and avoid dual booting Windows and Linux on the same drive if you can, as that removes both the extended partition nonsense and the need to resize to fill the drive after the event. If you really have to, then this method seems to work if you later decide to remove Windows - it just requires a sensible sized second drive, a live CD/USB, and a bunch of time.
