在闲置的Redmi 5 plus上弄了postmarketOS来玩，触摸屏驱动不行，结果弄完内核启动只会进lk2nd fastboot了。。又没有电脑，幸好留有recovery install zip，尝试在TWRP下面修复一下。  
用unzip解压安装包，得到`chroot`目录；用里面的工具把postmarketOS的分区映射出来。要用到的工具不是静态链接的，干脆直接chroot方便一点。  
参考postmarketOS recovery installer代码[pmos_install_functions](https://gitlab.com/postmarketOS/postmarketos-android-recovery-installer/-/blob/master/pmos_install_functions)，用kpartx映射分区，我安装postmarketOS的块设备是`/dev/block/by-name/userdata`，所以等下映射出来就是`/dev/mapper/userdata*`（和basename有关）：
```sh
mkdir chroot/dev && mkdir chroot/proc && mkdir chroot/sys
mount --bind /dev chroot/dev
mount --bind /proc chroot/proc
mount --bind /sys chroot/sys
chroot chroot bin/busybox sh
kpartx -afs /dev/block/by-name/userdata
exit
```
第一个分区是boot分区，我们给挂载上：  
```sh
mount -t ext2 /dev/mapper/userdata1 /data
```
尝试备份一下boot：  
```
# cp /data/ /system_root/
cp: Structure needs cleaning
```
看来文件系统有点问题。卸载卷，然后尝试修复一下：  
```sh
e2fsck -y /dev/mapper/userdata1
```
再挂载没问题了。重启试一下，不会进fastboot但是引导之后马上黑屏。看来我彻底把内核弄炸了，只好用安装包把boot覆盖一下。重来一次，这次把rootfs.tar.gz也解压出来，再把boot解压出来然后覆盖：  
```sh
tar -xzvf rootfs.tar.gz ./boot
rm -rf /data/*
cp -r /boot/* /data
umount /data
```
重启。ok这下进系统了，可能是dts问题没wifi，`doas apk fix linux-postmarketos-qcom-msm8963`然后从`/boot/dtbs`里面拿dtb覆盖一下`/boot`下面的就完美解决了。
