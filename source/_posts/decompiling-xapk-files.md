title: Decompiling XAPK Files
tags:
  - reversing
  - android
comments: true
date: 2016-02-28 07:41:03
---

While reviewing new Android reverse engineering questions on Stack Overflow, I came across this request to [decompile an `.xapk`](http://stackoverflow.com/questions/35070003/decompile-xapk-file). A brief, non-technical description of the format is described on [APKPure's website](https://apkpure.com/xapk.htm):

> XAPK is a brand new file format standard for Android APK package file. Contains all APK package and obb cache asset file to keep Android games or apps working, it always ends in ".xapk". To ensure games, applications run perfectly, APK Install one click install makes it easy for Android users directly install .apk, .xapk file to the root directory.
obb cache data?
<!-- more -->

The "OBB cache files" are usually pretty big for games and include all of the assets like maps, models, images, music, whatever.

Ok, so it looks like we have a new **APK** format specifically designed for games _and_ it comes with [its own installer](https://apkpure.com/xapk-installer/com.apkpure.installer). Since there's an installer, that might mean the actual `.apk` is encrypted and embedded in the OBB. Maybe there's some metadata in the `.xapk` that tells the installer how to lookup the decryption key from their servers? Maybe I'll need to search for the **ZIP** magic bytes and carve out the `.apk`?

Nope. _The original `.apk` is at the root of the `.xapk` archive._ EASY. (read: boring) Shit, I was looking for a challenge!

I downloaded and examined [Side Lift King](https://apkpure.com/side-lift-king/org.ammarz.MT).
![such lift, much king](/images/decompiling-xapk/side-lift-king.png)

Here's the shasum:
```bash
$ shasum Side\ Lift\ King_v2.0_apkpure.com.xapk
155dbed0809d49b477c6ab4c52f555cfc8f47144  Side Lift King_v2.0_apkpure.com.xapk
```

The `.xapk` is just a ZIP file:
```bash
$ file Side\ Lift\ King_v2.0_apkpure.com.xapk
Side Lift King_v2.0_apkpure.com.xapk: Zip archive data, at least v2.0 to extract
```

The original `org.ammarz.MT.apk` is just floating around:
```bash
$ unzip Side\ Lift\ King_v2.0_apkpure.com.xapk
Archive:  Side Lift King_v2.0_apkpure.com.xapk
 extracting: org.ammarz.MT.apk
 extracting: icon.png
 extracting: Android/obb/org.ammarz.MT/main.8.org.ammarz.MT.obb
 extracting: manifest.json
```

It's not encrypted or anything.
```
$ file org.ammarz.MT.apk
org.ammarz.MT.apk: Java archive data (JAR)
```

It decompiles fine with apktool:
```bash
$ apktool d org.ammarz.MT.apk
I: Using Apktool 2.0.1 on org.ammarz.MT.apk
I: Loading resource table...
I: Decoding AndroidManifest.xml with resources...
I: Loading resource table from file: /Users/caleb/Library/apktool/framework/1.apk
I: Regular manifest package...
I: Decoding file-resources...
I: Decoding values */* XMLs...
I: Baksmaling classes.dex...
I: Copying assets and libs...
I: Copying unknown files...
I: Copying original files...
```

The OBB file, in case you're curious, is just a **JAR** which contains lots of files in an `assets/` folder.
```bash
$ file Android/obb/org.ammarz.MT/main.8.org.ammarz.MT.obb                           1 ↵
Android/obb/org.ammarz.MT/main.8.org.ammarz.MT.obb: Java archive data (JAR)
```

The `manifest.json` must be the file used by the installer. It must be the metadata used by the installer app. Here is a pretty formatted version:
```json
{
   "xapk_version":1,
   "package_name":"org.ammarz.MT",
   "name":"Side Lift King",
   "version_code":"8",
   "version_name":"2.0",
   "min_sdk_version":"9",
   "target_sdk_version":"23",
   "permissions":[
      "android.permission.INTERNET",
      "android.permission.ACCESS_NETWORK_STATE",
      "android.permission.WAKE_LOCK",
      "android.permission.ACCESS_WIFI_STATE",
      "com.android.vending.CHECK_LICENSE",
      "android.permission.WRITE_EXTERNAL_STORAGE",
      "android.permission.READ_EXTERNAL_STORAGE"
   ],
   "total_size":58621386,
   "expansions":[
      {
         "file":"Android/obb/org.ammarz.MT/main.8.org.ammarz.MT.obb",
         "install_location":"EXTERNAL_STORAGE",
         "install_path":"Android/obb/org.ammarz.MT/main.8.org.ammarz.MT.obb"
      }
   ]
}
```

# Summary

If you want to decompile the `.xapk`, first unxip the `.apk` file floating around the root of the archive.
