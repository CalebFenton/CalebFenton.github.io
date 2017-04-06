title: Creating a Java VM from Android Native Code
tags:
  - research
  - android
  - jni
comments: true
date: 2017-04-05 11:11:11
---

If you're writing native / JNI code for Android, it's probably as native method of an Android app. These methods are always passed the Dalvik VM instance of the app as the first parameter. You need this to create `jstring`s and other Java objects, lookup classes and fields, etc. It's not normal for you to have to instantiate a VM from native code because most of the time, if you're using the Java Native Interface (JNI), you started in Java land and are only dipping into native code land for them sweet, sweet [http://www.androidauthority.com/java-vs-c-app-performance-689081/](performance benefits). However, if you're reverse engineering or writing an exploit, you're likely always delving int all kinds of unusual trouble which the developers reasonably believed would never happen or at least would only be a theoretical edge case.

I recently needed to create a VM from native code to pass Java object arguments to a JNI function. In this post, I want to share what I came up with and why I finally settled on this particular method.
<!-- more -->

## Standard Method

The official, standard method is documented here: [How to Create a JVM Instance in JNI](http://www.developer.com/java/data/how-to-create-a-jvm-instance-in-jni.html). Unfortunately, this _won't_ work in Android because `jint JNI_CreateJavaVM(JavaVM**, JNIEnv**, void*)` isn't exported. Even if you're not familiar with this function, it's name should be a [clue](https://media.giphy.com/media/l2Sq2AG0wrEXQgLGU/giphy.gif) that it's important. If you want to check if it's not exported yourself, look at `jni.h` from your Android NDK directory. In my case, it's in `android-sdk/android-ndk-r13b/platforms/android-9/arch-x86/usr/include/jni.h`. The relevant code is:

```c
#if 0  /* In practice, these are not exported by the NDK so don't declare them */
jint JNI_GetDefaultJavaVMInitArgs(void*);
jint JNI_CreateJavaVM(JavaVM**, JNIEnv**, void*);
jint JNI_GetCreatedJavaVMs(JavaVM**, jsize, jsize*);
#endif
```

If you try to compile the code, you'll probably get an error like this:

```
warning: implicit declaration of function 'JNI_CreateJavaVM' is invalid in C99
      [-Wimplicit-function-declaration]
```

The official documentation is still useful for understanding the API and what all those options and arguments do. But if we want to use this on Android, we're going to have to explicitly load the necessary methods from some library.

One useful detail this code shows is how to set the VM's class path. Here's how it's done:

```c
JavaVMOption jvmopt[1];
jvmopt[0].optionString = "-Djava.class.path=" + ".";
```

This sets the class path to the current directory (`.`). This is needed if you want your VM to access system or app classes. Some early experiments show that setting to a directory doesn't seem to work. I tried setting to `/data/local/tmp` and pushing naked *DEX* files, as well as a *JAR* containing *DEX* files and the app's *APK*. The *only* option that worked was setting the **full path** to either the *JAR*, *DEX*, or the *APK*. What was odd is that system classes (i.e. `java.lang.String`) were not accessible without having at least one valid file in the class path. In other words, `(*env)->FindClass(env, "java.lang.String")` returns `0` unless there's at least one file on the class path, even though `java.lant.String` is defined in the framework.

To test yourself, push an APK to the emulator or device:

```bash
adb push shim_app.apk /data/local/tmp
```

Use this for your `JavaVMOption`:

```c
JavaVMOption opt[2];
opt[0].optionString = "-Djava.class.path=/data/local/tmp/shim_app.apk";
opt[1].optionString = "-agentlib:jdwp=transport=dt_android_adb,suspend=n,server=y";
```

You should now be able to use `FindClass` to load system and app classes.

## UiccUnlock Method

My [genius cyber-sleuth skills](https://www.google.com/search?q=android+jni_createjavavm) revealed another possible technique. It's from [UiccUnlock.cpp](https://gist.github.com/tewilove/b65b0b15557c770739d6) which looks like a quasi-exploit to do a SIM unlock.

I won't claim to fully understand what it's doing, but the interesting part to me is in `get_transation_code`. Here's what it does:

* creates a Java VM
* use the VM to get reference to `com.android.internal.telephony.ITelephony$Stub` class
* get the `TRANSACTION_sendOemRilRequestRaw` field value
* destroy the VM
* return field value

It looks like the field value is used to check if the device is already unlocked or perhaps to check if the unlock method will work. I'm not sure. I just wanted to rip out the code to create the VM.

The approach seemed sound: load the relevant VM creation functions by through `libnativehelper.so` or `libdvm.so` as a backup. However, there were a few lines in there that looked weird:

```c
JniInvocation_ctor = (JniInvocation_ctor_t) dlsym(libnativehelper, "_ZN13JniInvocationC1Ev");
JniInvocation_dtor = (JniInvocation_dtor_t) dlsym(libnativehelper, "_ZN13JniInvocationD1Ev");
JniInvocation_Init = (JniInvocation_Init_t) dlsym(libnativehelper, "_ZN13JniInvocation4InitEPKc");
```

I couldn't find these functions documented anywhere. Someone really clever figured these out (kudos!). Without calling these functions, you get some weird errors:

```
W/dalvikvm(5395): No implementation found for native Landroid/os/SystemProperties;.native_get:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
W/dalvikvm(5395): No implementation found for native Landroid/os/SystemProperties;.native_get:(Ljava/lang/String;Ljava/lang/String;)Ljava/lang/String;
W/dalvikvm(5395): Exception Ljava/lang/UnsatisfiedLinkError; thrown while initializing Landroid/os/Build;
A/libc(5395): Fatal signal 11 (SIGSEGV) at 0x0000000c (code=1), thread 5395 (create_vm)
```

Despite these weird functions, this approach totally worked for me. But I wanted to understand what the `_ZN13JniInvocationC1Ev` functions did and if they were portable between Android versions. My gut told me that hardcoding that many function names which also included numbers could lead to incompatibilities in some devices or in later Android versions.

## Surfaceflinger Method

I eventually found code from Google's Surfaceflinger service: [DdmConnection.cpp](https://android.googlesource.com/platform/frameworks/native/+/ce3a0a5/services/surfaceflinger/DdmConnection.cpp).

This defaults to looking for `JNI_CreateJavaVM` in `libdvm.so`. Instead of calling `_ZN13JniInvocation*` methods, it looks like it calls `Java_com_android_internal_util_WithFramework_registerNatives` from `libandroid_runtime.so`. The behavior of `registerNatives` is described well [here](http://stackoverflow.com/questions/1010645/what-does-the-registernatives-method-do).

Also interesting are the options used to create the VM:

```c
opt.optionString = "-agentlib:jdwp=transport=dt_android_adb,suspend=n,server=y";
```

These options are well documented [here](http://www.netmite.com/android/mydroid/2.0/dalvik/docs/debugger.html) and according to the documentation, this just allows for debugging of the JVM. Seems pretty standard.

I noticed the JNI version was 1_4, but I bumped it up to 1_6 because that's what's used in the example code [here](https://developer.android.com/training/articles/perf-jni.html#native_libraries). Here are the supported versions from  `jni.h`:

```c
#define JNI_VERSION_1_1 0x00010001
#define JNI_VERSION_1_2 0x00010002
#define JNI_VERSION_1_4 0x00010004
#define JNI_VERSION_1_6 0x00010006
```

I ended up using this approach since I figured it's the most robust and future-proof because it comes from Google.

## Final Code

Here's the final code to create the VM:

```c
#include <dlfcn.h>
#include <jni.h>

typedef int (*JNI_CreateJavaVM_t)(void *, void *, void *);
typedef jint (*registerNatives_t)(JNIEnv* env, jclass clazz);

static int init_jvm(JavaVM **p_vm, JNIEnv **p_env) {
  // https://android.googlesource.com/platform/frameworks/native/+/ce3a0a5/services/surfaceflinger/DdmConnection.cpp
  JavaVMOption opt;
  opt.optionString = "-agentlib:jdwp=transport=dt_android_adb,suspend=n,server=y";

  JavaVMInitArgs args;
  args.version = JNI_VERSION_1_6;
  args.options = &opt;
  args.nOptions = 1;
  args.ignoreUnrecognized = JNI_FALSE;

  void *libdvm_dso = dlopen("libdvm.so", RTLD_NOW);
  void *libandroid_runtime_dso = dlopen("libandroid_runtime.so", RTLD_NOW);

  if (!libdvm_dso || !libandroid_runtime_dso) {
    return -1;
  }

  JNI_CreateJavaVM_t JNI_CreateJavaVM;
  JNI_CreateJavaVM = (JNI_CreateJavaVM_t) dlsym(libdvm_dso, "JNI_CreateJavaVM");
  if (!JNI_CreateJavaVM) {
    return -2;
  }

  registerNatives_t registerNatives;
  registerNatives = (registerNatives_t) dlsym(libandroid_runtime_dso, "Java_com_android_internal_util_WithFramework_registerNatives");
  if (!registerNatives) {
    return -3;
  }

  if (JNI_CreateJavaVM(&(*p_vm), &(*p_env), &args)) {
    return -4;
  }

  if (registerNatives(*p_env, 0)) {
    return -5;
  }

  return 0;
}
```


Here's how it's used:

```c
#include <stdlib.h>
#include <stdio.h>

JavaVM * vm = NULL;
JNIEnv * env = NULL;
int status = init_jvm( & vm, & env);
if (status == 0) {
  printf("Initialization success (vm=%p, env=%p)\n", vm, env);
} else {
  printf("Initialization failure (%i)\n", status);
  return -1;
}

jstring testy = (*env)->NewStringUTF(env, "this should work now!");
const char *str = (*env)->GetStringUTFChars(env, testy, NULL);
printf("testy: %s\n", str);
```

There you go. Now you have everything you need to instantiate a Java VM from native code.