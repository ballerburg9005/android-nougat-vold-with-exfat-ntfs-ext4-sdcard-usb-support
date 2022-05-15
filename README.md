# about

Chances are you found this repo, because you need to have a storage medium with large file support, but your Android version is not Nougat (v8). In this case the following explanation will still be helpful to understand what is going on in Android in the background and to decide how to proceed best.

When you insert an SD card or USB storage via OTG that is not formatted FAT32, then Android will report this media as corrupted. This is because the mounting of those media is handled via vold (Volume Manager), and vold supports only FAT32 up to Android version 8. From version 9-12 (and beyond?) it additionally supports exFAT, but still no other filesystem. ExFAT support in vold only works though, if the vendor actually compiled the FUSE binaries for it on the system partition. Since exFAT is kind of an oddball filesystem, unlike NTFS or ext4, it could very well be missing in your Android ROM. You can check the status of vold filesystem support [here](https://android.googlesource.com/platform/system/vold/+/refs/tags/android-mainline-12.0.0_r96/model/PublicVolume.cpp#103) (decent into vold to see different branches).

Diagram of removable media mounting on Android: [SD CARD or USB Harddrive] -> Vold -> FUSE binary -> Kernel

Like you can see, there are two points of failure: Vold and FUSE binaries. We already know that vold totally sucks, so the only way to change this is to recompile vold from Android source (see "compiling vold"). If you are using Android 7 or 8 with ARM64, then use the binaries from this repo for version 7 and for version 8 use https://github.com/null4n/vold-posix .

TBC
