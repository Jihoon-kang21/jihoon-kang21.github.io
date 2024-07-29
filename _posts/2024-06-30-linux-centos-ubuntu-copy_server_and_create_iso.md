---
layout: post
title: Linux - Ubuntu & CentOS - 서버 복사해서 그대로 설치하는 iso 만들기
subtitle: custom iso 만들기 스크립트
categories: Tech
tags: [linux,centos,ubuntu]
---

## 필요한 상황

- 똑같은 하드웨어에 똑같은 프로그램을 매번 설치해야하는 상황
- 필요한 모든 패키지를 설치한 상태의 서버를 복사에서 OS 설치 iso에 포함시킨다
- iso로 OS 설치하는 대신 복사한 서버의 내용이 붙여넣기 된다.
- 파일만 복사하면 안되고 스크립트를 이용해서 파티션을 나눈다음 복제했던 서버내용을 붙여넣기 한다.
- 완성한 iso 를 이용해서 부팅하면 복제했던 서버와 똑같은 상태가 된다.

## 만드는 법

centOS8 부팅 ISO를 마운트해서 server.tar.gz 와 install.sh를 복사하고 anaconda파일을 수정한다.

### 0. tar 파일 만들기

- docker, 제품 등 모든 것이 설치된 서버에서 아래 디렉토리들을 압축하여 복사해 놓는다.

proc, sys, run 빼고

```
# cd /
# tar -zcvf server.tar.gz boot bin dev etc home lib lib32 lib64 libx32 media mnt opt root sbin snap srv tmp usr var
```


### 1. ISO mount

- centOS 부팅용 iso 파일에 복제한 서버 내용을 포함 시킬 것이다.

```
# mount CentOS-Stream-8-x86_64-20230209-boot.iso centosiso_mounted
```


### 2. 새 폴더로 복사

- mount 해서 iso 내부 파일을 수정할 수 없다. 새로운 폴더를 만들어 iso를 재생성할 것이다.

```
# mkdir centosiso_copied
# cp -a centosiso_mounted/* centosiso_copied/
```


### 3. install.img 마운트하기

```
# mount -t squashfs centosiso_mounted/images/install.img install_mounted/
```


### 5. rootfs mount

```
# mount install_mounted/LiveOS/rootfs.img rootfs_mounted/
```


### 6. rootfs.img 만들기

- rootfs.img 아래 복제했던 서버 내용을 포함시킬 것이다.
- seek의 크기는 복제한 서버 용량보다 커야한다.

```
# dd if=/dev/zero of=rootfs.img bs=1024k seek=33000 count=0
# mkfs -t ext4 rootfs.img
# mkdir rootfs_copied/
# mount -o loop rootfs.img rootfs_copied
# cp -a rootfs_mounted/* rootfs_copied/
```


### 7. rootfs_copied 내용 수정

- anaconda, install.sh 파일을 작성한다 - 본문 아래에서 스크립트 내용 참고
- anaconda는 부팅할때 실행되는 스크립트이다. anaconda가 install.sh 파일을 실행하도록 파일을 작성한다
- install.sh 는 설치 iso 대신 파티션을 나누고 복제한 서버 압축파일을 복사 및 압축해제하는 역할을 한다.

### anaconda

```bash
#!/bin/bash
echo 'Begin the installation process'
sleep 3
bash install.sh
echo 'Complete'
echo 'Rebooting..'
sleep 3
#reboot
```


### install.sh

```bash
#!/bin/bash
echo 'install.sh is running'
sleep 3


# Function to stop RAID arrays on a disk
stop_raid_arrays() {
  local disk="$1"
  echo "Stopping RAID arrays on ${disk}..."
  for raid_device in $(mdadm --detail --scan | awk '{print $2}' | sed "s/md\//md/"); do
    mdadm --stop "${raid_device}"
  done
}

# Function to unmount all partitions on a disk
unmount_partitions() {
  local disk="$1"
  echo "Unmounting all partitions on ${disk}..."
  for partition in $(lsblk -l -o NAME,MOUNTPOINT | grep -E "^${disk}[0-9]+" | awk '{print $1}'); do
    mountpoint=$(lsblk -l -o NAME,MOUNTPOINT | grep "^${partition}" | awk '{print $2}')
    if [ -n "${mountpoint}" ]; then
      umount "${mountpoint}"
    fi
      done
}

# Function to delete all partitions from a disk
delete_partitions() {
  local disk="$1"
  echo "Deleting all partitions on ${disk}..."
  parted -s "${disk}" mklabel gpt
  echo "All partitions on ${disk} deleted."
}

# List all available block devices, filter out non-disk devices and USB drives
disks=$(lsblk -d -o NAME,TYPE,TRAN | grep -w disk | grep -v usb | awk '{print $1}')

raid=$(lsblk -l -o TYPE | grep -c raid)

echo "raid partition exist, will be deleted"
# Loop through all disks and unmount and delete their partitions
for disk in $disks; do
  device_path="/dev/${disk}"
  unmount_partitions "${disk}"
  delete_partitions "${device_path}"
  stop_raid_arrays "${device_path}"
done
fdisk -w always -W always /dev/nvme0n1 <<EOF
o
n
p
1

+50M
t
c
n
p
2

+1G
n
p
3

+50G
n
p


w
EOF

fdisk -w always -W always /dev/nvme1n1 <<EOF
o
n
p
1

+50M
t
c
n
p
2

+1G
n
p
3

+50G
n
p


w
EOF

echo "Created partitions"
lsblk
sleep 3


mdadm --create /dev/md0 --level=1 --raid-devices=2 --metadata=0.90 /dev/nvme0n1p1 /dev/nvme1n1p1
mdadm --create /dev/md1 --level=1 --raid-devices=2 --metadata=0.90 /dev/nvme0n1p2 /dev/nvme1n1p2
mdadm --create /dev/md2 --level=1 --raid-devices=2 --metadata=0.90 /dev/nvme0n1p3 /dev/nvme1n1p3
mdadm --create /dev/md3 --level=1 --raid-devices=2 --metadata=0.90 /dev/nvme0n1p4 /dev/nvme1n1p4

echo "Created RAID partitions"
lsblk
sleep 3

mkfs.vfat -n "EFI" /dev/md0
mkfs.xfs -f /dev/md1
mkfs.xfs -f /dev/md2
mkfs.xfs -f /dev/md3

echo "Formatted partitions"
df -h
sleep 3

mkdir /rfs
mount -t xfs /dev/md3 /rfs

cd /root
pwd
echo "In /root dir"
df -h

tar xvf server.tar.gz -C /rfs
echo "untar done"

cd /rfs
cp -r /rfs/boot /rfs/boot2
mount /dev/md1 /rfs/boot
cp -r /rfs/boot2/* /rfs/boot/
rm -rf /rfs/boot2

cp -r /rfs/var/log /rfs/var/log2
mount /dev/md2 /rfs/var/log2
cp -r /rfs/var/log2/* /rfs/var/log/
rm -rf /var/log2

mkdir -p /rfs/sys /rfs/proc /rfs/run

mount --bind /dev /rfs/dev
mount --bind /sys /rfs/sys
mount --bind /proc /rfs/proc
mount --bind /run /rfs/run


mount /dev/md0 /rfs/boot/efi

echo "mount done"


cat << EOF | chroot .
update-grub

grub-install --target=x86_64-efi --efi-directory=/boot/efi --removable --boot-directory=/boot --bootloader-id=grub

echo "/dev/disk/by-uuid/$(blkid /dev/md0 -s UUID -o value) /boot/efi vfat defaults 0 1" > /etc/fstab
echo "/dev/disk/by-uuid/$(blkid /dev/md1 -s UUID -o value) /boot xfs defaults 0 1" >> /etc/fstab
echo "/dev/disk/by-uuid/$(blkid /dev/md2 -s UUID -o value) /var/log xfs defaults 0 1" >> /etc/fstab
echo "/dev/disk/by-uuid/$(blkid /dev/md3 -s UUID -o value) / xfs defaults 0 1" >> /etc/fstab


exit
EOF
```


### 작성한 스크립트 복사하기

```
# cp anaconda rootfs_copied/usr/sbin/anaconda
# cp install.sh server.tar.gz rootfs_copied/root/
```


### 8. install.img 만들기

- 다시 만든 rootfs.img 를 포함하는 install.img를 새로 만든다

```
# umount rootfs_copied

# mkdir install_copied
# cp -a install_mounted/* install_copied/
# rm install_copied/LiveOS/rootfs.img
# cp rootfs.img install_copied/LiveOS/

# mksquashfs ./install_copied install.img
```


### 9. install.img 복사

```
# rm centosiso_copied/images/install.img
# mv install.img centosiso_copied/images/
```


### 10. iso 만들기

### mkiso.sh

```bash
#!/bin/bash

LABEL=CentOS-Stream-8-x86_64-dvd
ISO_FILE=/home/lunit/custom-iso/ubuntu-test.iso

mkisofs -J -R -l -D -L \
    -allow-limited-size \
        -c isolinux/boot.cat \
        -o "$ISO_FILE" \
        -b isolinux/isolinux.bin \
        -no-emul-boot -boot-load-size 4 -boot-info-table \
        -eltorito-alt-boot \
        -e images/efiboot.img -graft-points \
        -no-emul-boot \
        -V "$LABEL" /home/lunit/custom-iso/centosiso_copied


isohybrid --uefi $ISO_FILE
```


## 위 작업 이후 새로운 서버에 대한 iso를 만들 때

```
mount -o loop rootfs.img rootfs_copied/
```


### copy.sh

```bash
#!/bin/bash

SERVER=5820-cxr3151mmg1173-cc75
TAR=$SERVER.tar.gz
echo $SERVER

mount CentOS-Stream-8-x86_64-20230209-boot.iso centosiso_mounted
mount -t squashfs centosiso_mounted/images/install.img install_mounted/
mount install_mounted/LiveOS/rootfs.img rootfs_mounted/

if [ -f "./rootfs.img" ]; then
  rm -f rootfs.img
else
  echo "rootfs.img does not exist"
fi

dd if=/dev/zero of=rootfs.img bs=1024k seek=34000 count=0
mkfs -t ext4 rootfs.img
mount -o loop rootfs.img rootfs_copied
cp -a rootfs_mounted/* rootfs_copied/

mount -o loop rootfs.img rootfs_copied/

if [ -f "./rootfs_copied/root/install.sh" ]; then
  rm -f ./rootfs_copied/root/install.sh
fi

cp install.sh rootfs_copied/root/install.sh

if [ -f "./rootfs_copied/root/*.tar.gz" ]; then
  rm -f ./rootfs_copied/root/*.tar.gz
fi

cp $TAR rootfs_copied/root/server.tar.gz

rm -f rootfs_copied/usr/sbin/anaconda
cp anaconda rootfs_copied/usr/sbin/anaconda

umount rootfs_copied

#copy rootfs.img to install_copied directory
rm -f install_copied/LiveOS/rootfs.img
cp rootfs.img install_copied/LiveOS/

#create install.img using install_copied
if [ -f "./install.img" ]; then
  rm -f ./install.img
fi
mksquashfs ./install_copied install.img

#copy install.img to centosiso_copied directory
rm -f centosiso_copied/images/install.img
cp install.img centosiso_copied/images/


#make iso
LABEL=CentOS-Stream-8-x86_64-dvd
ISO_FILE=/home/lunit/custom-iso/ubuntu20-$SERVER.iso

mkisofs -J -R -l -D -L \
	-allow-limited-size \
        -c isolinux/boot.cat \
        -o "$ISO_FILE" \
        -b isolinux/isolinux.bin \
        -no-emul-boot -boot-load-size 4 -boot-info-table \
        -eltorito-alt-boot \
        -e images/efiboot.img -graft-points \
        -no-emul-boot \
        -V "$LABEL" /home/lunit/custom-iso/centosiso_copied


isohybrid --uefi $ISO_FILE
```


## 참고

https://hostingcontroller.com/english/support/HC-Virtualization-Module/WebHelp/HC-Templates/Xen-Templates/Creating_Disk_Image_for_Root_Filesystem.htm
https://elinux.org/Squash_FS_Howtof
https://www.suse.com/support/kb/doc/?id=000018770
parted efi 만드는 방법. mxlinux esp 설치 방법 parted 사용법 : https://m.cafe.daum.net/candan/ASbR/80
HowTo Create A GPT Disk With EFI System And exFAT Partitions Using Parted : https://wiki.networksecuritytoolkit.org/nstwiki/index.php?title=HowTo_Create_A_GPT_Disk_With_EFI_System_And_exFAT_Partitions_Using_Parted
RHEL/CentOS 7 Boot Process & Troubleshooting Tips : https://www.linkedin.com/pulse/rhelcentos-7-boot-process-troubleshooting-tips-jason-hensley/?articleId=6542387471525781504
https://linuxlink.timesys.com/docs/engineering/wiki/HOWTO_Install_GRUB2_with_EFI_support
live cd custom하기  : https://coupainus.tistory.com/3
How does the CentOS installation work from the inside? : https://unix.stackexchange.com/questions/80376/how-does-the-centos-installation-work-from-the-inside
fdisk 로 sw raid 구성하기 : https://hbesthee.tistory.com/2027
test 할 centos8 iso - 이건 설치할떄 부팅용. 이 안에 tar를 담을 예정 : http://ftp.kaist.ac.kr/CentOS/8-stream/isos/x86_64/CentOS-Stream-8-x86_64-20230209-boot.iso