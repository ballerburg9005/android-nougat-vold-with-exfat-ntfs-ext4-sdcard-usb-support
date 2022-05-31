# ❌broken abandoned project❌

I guess this approach will not work for a lot of people because vold itself is a piece of shit so that it might require lots of mysterious changes by custom vendor code. I copied this code from stock Android and I had to do a couple of fixes, in places in which my vendor vold binary would dump the same ignored error messages and on top extra debug messages not part of stock code. So end of story is that stock vold is so crappy and broken by design that without having source code of adapted vendor vold it is hardly feasible to get something viable going.

## about 

* requires root

Chances are you found this repo, because you need to have a storage medium with large file support, but your Android version is not Nougat (v8). In this case the following explanation will still be helpful to understand what is going on in Android in the background and to decide how to proceed best.

When you insert an SD card or USB storage via OTG that is not formatted FAT32, then Android will report this media as corrupted. This is because the mounting of those media is handled via vold (Volume Manager), and vold supports only FAT32 up to Android version 8. From version 9-12 (and beyond?) it additionally supports exFAT, but still no other filesystem. ExFAT support in vold only works though, if the vendor actually compiled the FUSE binaries for it on the system partition. Since exFAT is kind of an oddball filesystem, unlike NTFS or ext4, it could very well be missing in your Android ROM. You can check the status of vold filesystem support [here](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-mainline-12.0.0_r96/model/PublicVolume.cpp#103) (decent into vold to see different branches).

Diagram of removable media mounting on Android: [SD CARD or USB Harddrive] -> Vold -> FUSE binary -> Kernel

Like you can see, there are two points of failure: Vold and FUSE binaries. We already know that vold totally sucks, so the only way to change this is to recompile vold from Android source (see "compiling vold"). If you are using Android 7 or 8 with ARM64, then use the binary from this repo for version 7 and for version 8 use https://github.com/null4n/vold-posix . The same goes for FUSE binaries, which are available in the "system" folder. You can also do some kind of hack using bash scripts... which I recommend you rather try than this. 

## summary

**Do bash script hack, don't do this approach.**

This approach:

* Android = 7: This repo
* Android = 8: https://github.com/null4n/vold-posix
* Android > 8: If exFAT does not already work, use binaries from this repo but not vold binary (only exFAT will work).

Other approaches:

* ❌ Using apps from Google Play (many issues: cost money, people report corrupted filesystems, work either not on SD card or require root but are bugged abandonware. Problem is DIY implementation of FS in userspace = bad.)
* ❌ [sdcardfs.ko patch](https://forum.xda-developers.com/t/ntfs-on-your-android-with-full-read-write-support.2920856/) (I believe this is an approach that stopped working in Android 7, plus the last 10 years tons of phones now ship without loadable module support)
* ❌ [NtfsMounter.apk](https://forum.xda-developers.com/t/app-ntfs-mounter-automatically-mount-ntfs-ext-formated-usb-sticks-and-sd-cards.1654024/) can't deal with post Android 6 permissions, then just segfaults when you use workarounds: hopeless

## compiling vold

Beware that the download takes several hours, but the compilation is actually very easy and done in a few minutes.

```
# List of Android branches: 
mkdir ANDROID; cd ANDROID
# you need about 30GB disk space
repo init  --depth=1 -u https://android.googlesource.com/platform/manifest -b android-7.0.0_r31
repo sync -c -j8
source build/envsetup.sh && lunch
# pick aosp_arm64-eng - besides architecture, those flavors are meant for vendors to adapt to their product
# for me there was a bug, so that when I selected #3 it was actually #2 ...
mv system/vold system/vold_original
cp $thisrepo/vold system/vold 
cd system/vold
mma # compile command
cp ANDROID/out/target/product/generic_arm64/system/bin/vold $thisrepo/binaries/bin/
```

## installing vold and the other binaries it wants

```
adb push $thisrepo/binaries/ /sdcard/
adb shell
su
# it is best to perform this from recovery, or instantaneously. I actually had my system partition damaged by leaving it rw for too long.
mount -o rw,remount /system 
mv /system/bin/vold /system/bin/vold_bkup
cp /sdcard/binaries/bin/* /system/bin/
cp /sdcard/binaries/lib64/* /system/bin/
chmod 755 /system/bin/vold /system/bin/fsck.exfat /system/bin/fsck.ntfs /system/bin/mkfs.exfat /system/bin/mkfs.ntfs /system/bin/mount.exfat /system/bin/mount.ntfs
 mount -o ro,remount /system
```

## testing the new vold
```
# vold launch command, only needs you to add whitespaces
cat /proc/`ps | grep vold | sed -E "s#^[^[:space:]]+[[:space:]]+([^[:space:]]+).*#\1#g"`/cmdline | tr '\000' ' '
killall vold
echo "use above cmdline here"
sleep 5
ps | grep vold
echo "if you don't see that vold is still running, you need to check why with adb logcat and recompile it"
echo "without vold working, Android will not boot and you need to fix phone with TWRP or fastboot & stock image!"
```

## more debugging
```
# Once vold is gone, you can't push via adb to fix:
setenforce 0
mount /dev/block/mmcblk1p1 /sdcard/ -o umask=0000

# The last I got was :
# 05-31 16:19:24.721  6877  6877 E SocketListener: Obtaining file descriptor socket 'vold' failed: No such file or directory
# 05-31 16:19:24.721  6877  6877 E vold_bkup: Unable to start CommandListener: No such file or directory
# which I would also get with original vold, which would no longer function after I ran my vold

# then I forgot to mount ro, tried to copy vold_backup back but system partition was broken so I flashed it again
# then installed vold again without testing
# then rebooted
# then I gave up!
```
