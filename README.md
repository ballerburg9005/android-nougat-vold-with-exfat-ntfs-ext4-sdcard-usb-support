# about

Chances are you found this repo, because you need to have a storage medium with large file support, but your Android version is not Nougat (v8). In this case the following explanation will still be helpful to understand what is going on in Android in the background and to decide how to proceed best.

When you insert an SD card or USB storage via OTG that is not formatted FAT32, then Android will report this media as corrupted. This is because the mounting of those media is handled via vold (Volume Manager), and vold supports only FAT32 up to Android version 8. From version 9-12 (and beyond?) it additionally supports exFAT, but still no other filesystem. ExFAT support in vold only works though, if the vendor actually compiled the FUSE binaries for it on the system partition. Since exFAT is kind of an oddball filesystem, unlike NTFS or ext4, it could very well be missing in your Android ROM. You can check the status of vold filesystem support [here](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-mainline-12.0.0_r96/model/PublicVolume.cpp#103) (decent into vold to see different branches).

Diagram of removable media mounting on Android: [SD CARD or USB Harddrive] -> Vold -> FUSE binary -> Kernel

Like you can see, there are two points of failure: Vold and FUSE binaries. We already know that vold totally sucks, so the only way to change this is to recompile vold from Android source (see "compiling vold"). If you are using Android 7 or 8 with ARM64, then use the binary from this repo for version 7 and for version 8 use https://github.com/null4n/vold-posix . The same goes for FUSE binaries, which are available in the "system" folder.

# compiling vold

Beware that the download takes several hours, but the compilation is actually very easy and done in a few minutes.

```
# List of Android branches: 
mkdir ANDROID; cd ANDROID
# you need about 30GB disk space
repo init  --depth=1 -u https://android.googlesource.com/platform/manifest -b android-7.0.0_r31
repo sync -c -j8
source build/envsetup.sh
lunch
# pick aosp_arm64-eng - besides architecture, those flavors are meant for vendors to adapt to their product
mv system/vold system/vold_original
cp $thisrepo/vold_src system/vold 
cd system/vold
mma # compile command
cp ANDROID/out/target/product/generic_arm64/system/bin/vold $thisrepo/binaries/bin/
```

# installing vold and the other binaries it wants

```
adb push $thisrepo/binaries/ /sdcard/
adb shell
su
mount -o rw,remount /system 
cp /sdcard/binaries/bin/* /system/bin/
cp /sdcard/binaries/lib64/* /system/bin/
mv /system/bin/vold /system/bin/vold_bkup
chmod 755 /system/bin/vold /system/bin/fsck.exfat /system/bin/fsck.ntfs /system/bin/mkfs.exfat /system/bin/mkfs.ntfs /system/bin/mount.exfat /system/bin/mount.ntfs
 mount -o ro,remount /system # important to remount ro, or else vold might damage your system partition!
```

# testing the new vold
```
# vold launch command, only needs you to add whitespaces
cat /proc/`ps | grep vold | sed -E "s#^[^[:space:]]+[[:space:]]+([^[:space:]]+).*#\1#g"`/cmdline
# killall vold
echo "use above cmdline here"
sleep 5
ps | grep vold
echo "if you don't see that vold is still running, you need to check why with adb logcat and recompile it"
echo "without vold working, Android will not boot and you need to fix phone with TWRP or fastboot & stock image!"
```
