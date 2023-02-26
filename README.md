# Emulating Raspberry Pi with Qemu
Linux version:
```
$ lsb_release -a
Distributor ID: Ubuntu
Description:    Ubuntu 20.04.4 LTS
Release:        20.04
Codename:       focal
```
## Preparing Files
Download and build Qemu-7.2.0:

```bash
# Dependent library
$ apt-get install libslirp-dev

# Qemu
$ wget https://download.qemu.org/qemu-7.2.0.tar.xz
$ wget https://download.qemu.org/qemu-7.2.0.tar.xz
$ tar xvJf qemu-7.2.0.tar.xz
$ cd qemu-7.2.0
$ ./configure
$ sudo make & make install
```

Create workplace:

```bash
$ mkdir ~/raspbian
```

Download qemu-rpi-kernel:

```bash
$ cd ~ && git clone https://github.com/dhruvvyas90/qemu-rpi-kernel.git
$ cp ~/qemu-rpi-kernel/kernel-qemu-4.14.79-stretch ~/raspbian/
$ cp ~/qemu-rpi-kernel/versatile-pb.dtb ~/raspbian/
$ rm -rf ~/qemu-rpi-kernel
```

Download Raspbian disk image and unzip:

```bash
$ cd ~/raspbian/
$ wget http://downloads.raspberrypi.org/raspbian/images/raspbian-2020-02-14/2020-02-13-raspbian-buster.zip
$ unzip 2020-02-13-raspbian-buster.zip && rm 2020-02-13-raspbian-buster.zip
$ mv 2020-02-13-raspbian-buster.img raspbian.img
```

Files in the current directory:

```bash
$ ls ~/raspbian/
kernel-qemu-4.14.79-stretch raspbian.img versatile-pb.dtb
```

## Edit Raspbian Image File

Find starting sector of the image's second partition:

```bash
$ fdisk -l raspbian.img
Disk raspbian.img: 3.54 GiB, 3787456512 bytes, 7397376 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0xea7d04d6

Device        Boot  Start     End Sectors  Size Id Type
raspbian.img1        8192  532479  524288  256M  c W95 FAT32 (LBA)
raspbian.img2      532480 7397375 6864896  3.3G 83 Linux
```

Multiply that sector number by 512 and use the result as the offset in mounting the image:

```bash
$ mkdir /mnt/raspbian

# 532480*512 = 272629760
$ sudo mount ./raspbian.img -o offset=272629760 /mnt/raspbian
```

Edit the ld.so.preload file to comment:

```bash
$ sudo vi /mnt/raspbian/etc/ld.so.preload

# comment the only line
/usr/lib/arm-linux-gnueabihf/libarmmem-${PLATFORM}.so

$ sudo umount /mnt/raspbian
```

Convert the raw img file to qcow2 format:

```bash
$ qemu-img convert -f raw -O qcow2 ./raspbian.img raspbian.qcow2
```

## Starting the virtual machine

Copy the following code into a startup script called something like "start.sh":

```bash
#！/usr/bin/env bash 
sudo qemu-system-arm -kernel kernel-qemu-4.14.79-stretch \
-cpu arm1176 -m 256 \
-M versatilepb -dtb versatile-pb.dtb \
-no-reboot \
-serial stdio \
-append " root=/dev/sda2 panic=1 rootfstype=ext4 rw " \
-hda raspbian.qcow2 \
-net nic \
-net user,hostfwd=tcp::5555-:22
```

Starting the virtual machine:

```bash
$ ./start.sh
...
raspberrypi login:pi
Password:raspberry
...

pi@raspberrypi:~$ 
```

## Expand the disk size

The shipping image is 91% full:

```bash
pi@raspberrypi:~$ df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/root      ext4      3.1G  2.8G  0.3G  91% /
...
```

Shut down the virtual machine:

```bash
pi@raspberrypi:~$ sudo shutdown -h now
```

Increase the disk size:

```bash
$ qemu-img resize rpitest.qcow2 +4G
$ ./start.sh
...

pi@raspberrypi:~$ sudo sed -E 's/mmcblk0p?/sda/' /usr/bin/raspi-config | sudo bash
# Select: (7) Advanced options -> (A1) Expand Filesystem -> <Finish>
# Reboot now: <Yes>
```

Confirm the expanded disk size is recognized by the system:

```bash
pi@raspberrypi:~$ df -hT
Filesystem     Type      Size  Used Avail Use% Mounted on
/dev/root      ext4      7.1G  2.8G  4.1G  41% /
...
```

## Enabling SSH on Raspberry Pi

```bash
pi@raspberrypi:~$ sudo raspi-config
# (5)Interfacing Options -> (P2)SSh -> enable ssh? <yes>

pi@raspberrypi:~$ service ssh status
● ssh.service - OpenBSD Secure Shell server
   Loaded: loaded (/lib/systemd/system/ssh.service; enabled; vendor preset: enab
   Active: active (running) since Sun 2023-02-26 07:27:52 GMT; 39s ago
...
```

```bash
$ ssh pi@127.0.1.1 -p 5555
pi@127.0.1.1's password: raspberry
...
pi@raspberrypi:~ $ 
```

## References

[Emulating a Raspberry Pi with QEMU](https://gist.github.com/plembo/c4920016312f058209f5765cb9a3a25e)

[Emulating Raspberry Pi with QEMU](https://blog.ramdoot.in/emulating-raspberry-pi-on-with-qemu-951283daf2bd)

[RASPBERRY PI ON QEMU](https://azeria-labs.com/emulate-raspberry-pi-with-qemu/)

