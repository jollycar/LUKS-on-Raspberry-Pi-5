## LUKS on Raspberry Pi 5

***Last updated: April 2024.***

***NOTE: THIS GUID IS BEING UPDATED AND IS UNRELIABLE FOR NOW. USE AT YOUR OWN RISK***

***Instead of following this guide, I recommend using [sdm](https://github.com/jollycar/sdm) 
(forked from original repository: [gitbls](https://github.com/gitbls/sdm))***

***If you have an SDCARD inserted, make sure you don't have any operating systems installed. 
Either remove it or wipe partitions***

This guide explains how to encrypt the root partition of a nvme drive with Raspberry Pi OS (Debian GNU/Linux 12 (bookworm)) with LUKS. 
The process requires a Raspberry Pi 5 running Raspberry Pi OS (Debian GNU/Linux 12 (bookworm)) on the nvme drive 
and a USB memory with at lease the same capacity as the nvme drive. 

It is important to have a backup of the nvme drive, in case anything goes wrong, because it will be overwritten, and of the USB memory as well.

Full disk encryption is quite easy to perform with modern Linux distributions. 
Raspberry Pi is an exception because the boot partition does not include most of the needed programs and kernel modules. 
On the other hand, it is important to use disk encryption with Raspberry Pi, 
because the nvme drive can be extracted from the unit and its content read quite easily.

### Requirements

Linux kernel 5.0 or later. You can check it with this command:
```
uname -s -r
```
cryptsetup 2.0.6 or later. You can check it with this command:
```
cryptsetup --version
```
You should install the programs needed:
```
sudo apt install busybox cryptsetup initramfs-tools
```
The microprocessors of the Raspberry Pi 5 include AES acceleration (AES-256-CBC). The decryption performance should be around 1800 MiB/s.
You can check that every module is present and loaded with this command:
```
cryptsetup benchmark -c aes-cbc
```
The output, if everything is all right, will be like this:
```
# Tests are approximate using memory only (no storage IO).
#            Algorithm |       Key |      Encryption |      Decryption
               aes-cbc        256b       868.3 MiB/s      1805.9 MiB/s
```
If the execution shows an error message complaining about the cipher not being available, either the kernel modules are not present or are not loaded. 
You can load the necessary kernel modules for that cipher with ‘modprobe’:
```
sudo modprobe aes-arm64
sudo modprobe cbc
```
If ‘modprobe’ complains about the module not being found, the kernel module is not present. A possible cause can be that the kernel version is not 5.0 or later.

### Preparing Linux

‘initramfs’ has to be recreated when a new kernel is installed or just now that we have to change its configuration. We need to create a new file:
```
/etc/kernel/postinst.d/initramfs-rebuild
```
and it should have this content:
```bash
#!/bin/sh -e

# Rebuild initramfs.gz after kernel upgrade to include new kernel's modules.
# https://github.com/Robpol86/robpol86.com/blob/master/docs/_static/initramfs-rebuild.sh
# Save as (chmod +x): /etc/kernel/postinst.d/initramfs-rebuild

# Remove splash from cmdline.
if grep -q '\bsplash\b' /boot/firmware/cmdline.txt; then
  sed -i 's/ \?splash \?/ /' /boot/firmware/cmdline.txt
fi

# Exit if not building kernel for this Raspberry Pi's hardware version.
version="$1"
current_version="$(uname -r)"
case "${current_version}" in
  *-v7+)
    case "${version}" in
      *-v7+) ;;
      *) exit 0
    esac
  ;;
  *+)
    case "${version}" in
      *-v7+) exit 0 ;;
    esac
  ;;
esac

# Exit if rebuild cannot be performed or not needed.
[ -x /usr/sbin/mkinitramfs ] || exit 0
[ -f /boot/initramfs.gz ] || exit 0
lsinitramfs /boot/initramfs.gz |grep -q "/$version$" && exit 0  # Already in initramfs.

# Rebuild.
mkinitramfs -o /boot/initramfs.gz "$version"
```
The file should be made executable:
```
sudo chmod +x /etc/kernel/postinst.d/initramfs-rebuild
```
We also need to specify some programs that need to be included in ‘initramfs’. We do that with a new file at:
```
/etc/initramfs-tools/hooks/luks_hooks
```
and it should have this content:
```bash
#!/bin/sh -e
PREREQS=""
case $1 in
        prereqs) echo "${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec /sbin/resize2fs /sbin
copy_exec /sbin/fdisk /sbin
copy_exec /sbin/cryptsetup /sbin
```
The programs are ‘resize2fs’, ‘fdisk’ and ‘cryptsetup’.
The file should be made executable:
```
sudo chmod +x /etc/initramfs-tools/hooks/luks_hooks
```
‘initramfs’ for Raspberry Pi OS does not include kernel modules for LUKS and encryption by default. 
We need to configure the kernel modules to add. This file has to be edited:
```
/etc/initramfs-tools/modules
```
and the following lines with the names of kernel modules added:
```
algif_skcipher
aes_arm64
aes_ce_blk
aes_ce_ccm
aes_ce_cipher
sha256_arm64
cbc
dm-crypt
```
Now we need to build the new ‘initramfs’:
```
sudo -E CRYPTSETUP=y mkinitramfs -o /boot/initramfs.gz
```
You can ignore the warning about ‘cryptsetup’ if the next checking is correct.

We can check that the programs are present in ‘initramfs’ with the following command:
```
lsinitramfs /boot/initramfs.gz | grep -P "sbin/(cryptsetup|resize2fs|fdisk)"
```
We can check that the kernel modules are present in ‘initramfs’ with the following command:
```
lsinitramfs /boot/initramfs.gz | grep -P "(algif_skcipher|aes-arm64|sha256-arm64|cbc|dm-crypt)"
```

### Preparing Boot
We need to modify some files before rebooting the Rasperry Pi. These changes are meant to tell the boot process to use an encrypted root filesystem. 
After the previous changes, the Raspberry Pi will boot correctly. After changing the following files, 
the Raspberry Pi will not boot to Desktop until the whole process of encrypting the root partition and configuring LUKS is completed. 
If after modifying the next four files and before the root partition is encrypted anything goes wrong, 
the changes in the four files can be reverted and the Raspberry Pi would boot normally. It is a good idea to make a copy of those file before modifying them.

**File: /boot/firmware/config.txt**

The next line has to be appended at the end of the file:
```
initramfs initramfs.gz followkernel
```

**File: /boot/firmware/cmdline.txt**

It contains one line with parameters. One of them is ‘root’, that specifies the location of the root partition. 
For Raspberry Pi is usually ‘/dev/nvme0n1p2’, but it can also be other device (or the same) specified as “PARTUUID=xxxxx”. 
The value of ‘root’ has to be change to ‘/dev/mapper/cryptroot’. For example, if ‘root’ is:
```
root=/dev/nvme0n1p2
```
it should be changed to:
```
root=/dev/mapper/cryptroot
```
also, at the end of the line, separated by a space, this text should be appended:
```
cryptdevice=/dev/nvme0n1p2:cryptroot break=init
```

The 'break=init' is to force it to boot into the initramfs shell on the next boot. This change is not permanent; 
it will only affect the next boot if you remove break=init after use.

**File: /etc/fstab**

The device for root partition (‘/’) should be changed to the mapper.
For example, if the device for root is:
```
/dev/nvme0n1p2
```
it should be changed to:
```
/dev/mapper/cryptroot
```

**File: etc/crypttab**

At the end of the file a new line should be appended with the next content:
```
nvme	/dev/nvme0n1p2	none	luks
```

Everything is ready now to reboot. After rebooting, Raspberry Pi OS will fail to start because we have configured a root partition that does not exist yet. 
After several equal messages indicating the failure, the ‘initramfs’ shell will show.

### Encrypting the root partition

We have to copy the root partition of the nvme drive to the USB memory. 
The idea is to have a copy of the root partition of the nvme drive in the USB memory, 
create an encrypted volume in the root partition (the content will be lost) 
and copy the root partition in the USB memory back to the encrypted partition of the nvme drive. 
Take into account that the previous content of the USB memory will be lost. 
Before doing that, we are going to check the root partition and reduce the size of its filesystem to the minimum possible, 
so that the copy operation takes less time. The command to check it and correct possible errors is:
```
e2fsck -f /dev/nvme0n1p2
```
After finishing, we have to resize the filesystem. It is very important to take note of the size of the filesystem after resizing it, 
the number of blocks of 4k:
```
resize2fs -fM -p /dev/nvme0n1p2
```
An example of the output of the program is:
```
Resizing the filesystem on /dev/nvme0n1p2 to 2345678 (4k) blocks.
```
The number to take note of is ‘2345678’ in the example.
Once the resizing is finished we are going to get a checksum value of the whole partition. 
We will do the same after every copy operation between nvme drive and USB memory. If the checksum is the same, the copy will be equal. 
Let’s create the checksum for the root partition to copy. Remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem. 
It is useful to add “time ” before “dd” to obtain how long it takes the operation. When the rest of checksums are calculate it, 
it allows to know, more or less, how much time I will take:
```
time dd bs=4k count=XXXXX if=/dev/nvme0n1p2 | sha1sum
```
Take note of the SHA1 value. 
Connect now the USB memory to the Raspberry Pi. Some information about the USB memory will be printed. 
If there is no other USB memory connected, the device for the memory will probably be /dev/sda. It can be checked with:
```
fdisk -l /dev/sda
```
We are going to use the whole USB memory, /dev/sda.
This operation will copy the root filesystem of the nvme drive into the USB memory device, 
deleting all its content (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/nvme0n1p2 of=/dev/sda
```
While copying to or from nvme drives and USB memories a message might appear indicating that task worker has been block for several seconds. 
It can be ignored if the checksums are correct.

Now we have the original root filesystem and the copy. We also have the checksum of the original. 
We have to calculate the checksum of the copy and compare them. We can consider they are a exact copy if the checksums coincide. 
The command to calculate the checksum of the copy is (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sda | sha1sum
```
Assuming that the checksums are correct, now it is time to encrypt the root filesystem of the nvme drive, to create the LUKS volume using ‘cryptsetup’. 
There are many parameters and possible values for the encryption. This is the command I have chosen:
```
cryptsetup luksFormat --type luks2 --cipher aes-cbc-essiv:sha256 --hash sha256 --iter-time 5000 --key-size 256 --pbkdf argon2i /dev/nvme0n1p2
```
More information about the parameters can be found here:
<https://man7.org/linux/man-pages/man8/cryptsetup.8.html>

The command will ask for a passphrase twice (for confirmation). 
It is important that the passphrase is long and uses different characters sets (letter, numbers, symbols, uppercase, lower case, etc.).
After creating the LUKS volumen, we have to open it and copy the content of the root filesystem into it. The command to open the LUKS volume is:
```
cryptsetup luksOpen /dev/nvme0n1p2 nvme
```
It will ask the passphrase chosen in the previous stage. Once opened, we copy the root filesystem in the USB memory into the encrypted volume 
(remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/sda of=/dev/mapper/cryptroot
```
After copied, we have to calculate the checksum of the copy once more to validate it. If the checksums coincide, 
the copy is correct (remember to substitute ‘XXXXX’ with the number of 4k blocks you got after resizing the filesystem):
```
time dd bs=4k count=XXXXX if=/dev/mapper/cryptroot | sha1sum
```
In addition to the checksum check, we check the filesystem of the LUKS volume:
```
e2fsck -f /dev/mapper/cryptroot
```
We copied a reduced filesystem. Now we have to expand it to the size of the nvme drive:
```
resize2fs -f /dev/mapper/cryptroot
```
The process is nearly finished now. The USB memory can be extracted because is not needed anymore. We have to exit for the boot process to continue:
```
exit
```
The boot process will enter into initramfs shell again. At this moment we have to open the LUKS volume we just created to make the root filesystem available:
```
cryptsetup luksOpen /dev/nvme0n1p2 nvme
```
After opening the LUKS volumen we have to exit again and Raspberry Pi OS will start normally:
```
exit
```

### Booting
We do not want to enter into initramfs shell every time we switch on our Raspberry Pi. We can make Raspberry Pi OS ask for the password. 
What we need for that is building initramfs once more and reboot:
```
sudo mkinitramfs -o /tmp/initramfs.gz
sudo cp /tmp/initramfs.gz /boot/initramfs.gz
```

After rebooting, a prompt message should appear, something like “Please unlock disk...”. 
Perhaps the prompt asking for a password will get lost between some star-up messages, but you can enter your passphrase anyway and it should work.

