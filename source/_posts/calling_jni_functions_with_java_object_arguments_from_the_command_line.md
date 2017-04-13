title: Calling JNI Functions with Java Object Arguments from the Command Line
tags:
  - research
  - android
  - jni
comments: true
date: 2017-04-14 11:11:11
---

When analyzing malware or [penetration testing](http://i.imgur.com/hV9YDNn.gif) an app which uses a native library, it's helpful to isolate and execute the library's functions. This opens the door for debugging and using the malware's own code against it. For example, if the malware has encrypted strings and the decryption is done by a native function, you could either spend a bunch of time [reversing](http://imgur.com/gallery/9Q1fmSI) the algorithm to write your own decryption routine or you could just [harness](http://68.media.tumblr.com/f986bf831261e6935a59c18dccf73f17/tumblr_mltn4vLW8j1rn15aho1_500.gif) the function such that you can execute it with arbitrary inputs. If the malware author completely changes their decryption, you might not have to change anything. In this post, I'll explain how to harness a native library and execute its functions even if they require arguments from a live JVM instance.

In a previous post, I explained how to [create a Java VM from Android native code](/2017/04/05/creating_java_vm_from_android_native_code/) but I didn't give any real examples of how to use it. In this post, I'll give a concrete example.
<!-- more -->

There are at least two approaches to harness a native function. The first is to modify the app to accept some input from you and pass that to the native function. For example, you can write an intent filter, [convert it to Smali](https://calebfenton.github.io/2016/07/31/understanding_dalvik_static_fields_1_of_2/), add the code to the target app, modify the manifest, run the app, and send it intents via `adb` with your arguments. Even better, you could add a small socket or web server instead of an intent filter and send `curl` requests, which doesn't require modifying the manifest.

The second approach is to create a small native executable which loads the library, calls the target function, can be executed from the command line, and passes whatever arguments you give it. This makes it easier to debug since you're just running an executable rather than an entire app. 

## Target App

I created an example app so you can follow along at home. It's called [native-harness-target](https://github.com/CalebFenton/native-harness-target). To clone and build (of course replace `$ANDROID_*` vars for yourself):

```bash
git clone https://github.com/CalebFenton/native-harness-target.git
cd native-harness-target
echo 'ndk.dir=$ANDROID_NDK' > local.properties
echo 'sdk.dir=$ANDROID_SDK' >> local.properties
./gradlew build
```

The APKs will be in _app/build/outputs/apk/_. For this post, I'll be using an x86 emulator image and _app-universal-debug.apk_.

The app has an encrypted string and uses a native library to decrypt the string at run time. Here's how the string decryption looks in Smali:

```smali
const/16 v3, 0x57
new-array v1, v3, [B
fill-array-data v1, :array_2a

.local v1, "encryptedStringBytes":[B
invoke-static {}, Lorg/cf/nativeharness/Cryptor;->getInstance()Lorg/cf/nativeharness/Cryptor;
move-result-object v0

.line 21
.local v0, "c":Lorg/cf/nativeharness/Cryptor;

# v3 contains a String made from encrypted bytes
new-instance v3, Ljava/lang/String;
invoke-direct {v3, v1}, Ljava/lang/String;-><init>([B)V

# Call the decryption method, move result back to v3
invoke-virtual {v0, v3}, Lorg/cf/nativeharness/Cryptor;->decryptString(Ljava/lang/String;)Ljava/lang/String;
move-result-object v3
```

## Building the Harness

I started with a tool called [native-shim](https://github.com/rednaga/native-shim) by Tim "diff" Strazzere (a fellow [RedNaga](https://rednaga.io/) member!) as a foundation for the harness. What [shim](http://vibralign.com/wp-content/uploads/footshim.jpg) does is load a library and call its `JNI_OnLoad`. This makes debugging easy because you can just tell your debugger to start `shim` and pass the path to the target library as an argument. Set your debugger to break on library load and you can step through the `JNI_OnLoad`. Also, native-shim is great because it shows how to do almost everything you need to make harness work: load libraries (_.so_ files), get references to functions, and call them.

First, I added code to [initialize a Java VM instance](/2017/04/05/creating_java_vm_from_android_native_code/) and passed that instance to `JNI_OnLoad`. This makes for a more realistic JNI initialization. Without a real VM instance, the [internal state of the JNI library may be a little weird](https://cdn.meme.am/cache/instances/folder625/500x/50419625.jpg). It really depends on how `JNI_OnLoad` is implemented by the particular library. It may not matter at all, but it's common to [check the JNI version](https://developer.android.com/training/articles/perf-jni.html#native_libraries) from the little code I've seen, and to do that you need an instance of the VM.

```c
printf(" [+] Initializing JavaVM Instance\n");
JavaVM *vm = NULL;
JNIEnv *env = NULL;
int status = init_jvm(&vm, &env);
if (status == 0) {
  printf(" [+] Initialization success (vm=%p, env=%p)\n", vm, env);
} else {
  printf(" [!] Initialization failure (%i)\n", status);
  return -1;
}

printf(" [+] Calling JNI_OnLoad\n");
onLoadFunc(vm, NULL);
```

Tim, just let me know if you want this in native-shim and I'll send [a pull request](https://cdn.meme.am/instances/500x/62024625/please-guy-please-merge-my-pull-request.jpg).

Eventually, the goal was to make the harness open a socket server, read arguments over the socket, and call the function with those arguments. This way, the decryption function just becomes a service, and a Python script could easily interface with it.

### Understand the Target Function

To call a function you need the function signature and the return type. To get this, let's look at a decompilation of  `org.cf.nativeharness.Cryptor` which declares the `decryptString` native method.

```java
public class Cryptor {

    private static Cryptor instance = null;

    public static Cryptor getInstance() {
        if (instance == null) {
            instance = new Cryptor();
        }

        return instance;
    }

    public native String decryptString(String encryptedString);
}
```

From this code, you can see the method takes a `String` and returns a `String`. Seems simple. Let's convert that to a native function method signature.

```c
Java_org_cf_nativeharness_Cryptor_decryptString(JNIEnv *env, jstring encryptedString)
```

Every JNI native method needs `JNIEnv` as the first argument. This means the typedef for our function should be:

```c
typedef jstring(*decryptString_t)(JNIEnv *, jstring);
```

Unfortunately, if you try executing this function using the above typedef, you'll get a cryptic error message:

```
E/dalvikvm: JNI ERROR (app bug): attempt to use stale local reference 0x1
E/dalvikvm: VM aborting
A/libc: Fatal signal 6 (SIGABRT) at 0x00000a9a (code=-6), thread 2714 (harness)
```

This confused me for a while. I thought maybe I was getting a null reference somewhere so I added lots of `printf`s to show the memory locations of all the relevant pointers. The error really sounds like there's something wrong with one of the arguments, but all of the pointers looked good, none were null.

I had the idea to be extra super double sure I got the method signature right. Maybe there's some JNI boiler plate I was forgetting? To do this, I used [`javah`](http://docs.oracle.com/javase/7/docs/technotes/tools/windows/javah.html) which generates C header and source files that are needed to implement native methods.

To do this, you'll need [dex2jar](https://github.com/pxb1988/dex2jar) installed and on your class path and you need to change `platforms/android-19` to point to whichever platform you have installed.

```
$ d2j-dex2jar.sh app-universal-debug.apk
dex2jar app-universal-debug.apk -> ./app-universal-debug-dex2jar.jar
$ javah -cp app-universal-debug-dex2jar.jar:$ANDROID_SDK/platforms/android-19/android.jar org.cf.nativeharness.Cryptor
```

This creates _org\_cf\_nativeharness\_Cryptor.h_ which contains:

```c
JNIEXPORT jstring JNICALL Java_org_cf_nativeharness_Cryptor_decryptString
  (JNIEnv *, jobject, jstring);
```

This has `jobject` as the second argument. **WHY?** What gives? If already know the answer to this, I bet you've spent a lot of time looking at [Smali](https://github.com/github/linguist/pull/2422), specifically `invoke-virtual`. Whenever you call a virtual method (i.e. usually anything non-static), the first argument is _an instance of the object which implements the method_. In this case, the first argument should be an instance of `org.cf.nativeharness.Cryptor`.

Of course, you could cheat and just look at [_str-crypt.c_](https://github.com/CalebFenton/native-harness-target/blob/master/app/src/main/cpp/str-crypt.c) to find the signature but if you're really reverse engineering or pen-testing, you won't have the source.

The _real_ function typedef should have a `jobject` for the `Cryptor` instance as the first argument:

```c
typedef jstring(*decryptString_t)(JNIEnv *, jobject, jstring);
```

You may be wondering why the method is not static to begin with. There isn't a good reason for it to be static, true. But in the original app which made me write this blog, the target method wasn't static and I ran into this problem.

The lesson here is if you're not sure what the signature is, try `javah` and keep in mind virtual methods take an instance for the first argument, similar to Java's [`Method#invoke()`](https://docs.oracle.com/javase/7/docs/api/java/lang/reflect/Method.html#invoke(java.lang.Object,%20java.lang.Object...)).

## Building the Socket Server

This is the least interesting part of harness. If you don't mind, I'm just going to skip this. You can see [the code](https://github.com/CalebFenton/native-harness-target/blob/master/harness/server.c) for yourself. Also, I'm just a C tourist. If you think the code is shit, I believe you. But if you want to tell me it's shit, it must come with [a pull request](https://cdn.meme.am/cache/instances/folder117/500x/22899117.jpg).

## Using the Harness

Here's an overview of the steps required to test the harness:

1. start an emulator
* push the harness to the device
* push target native library and any dependencies to the device (in this case, there are no dependencies)
* push the native harness target app to the device
* start the harness
* forward ports from the emulator to the host
* run [_decrypt\_string.py_](https://github.com/CalebFenton/native-harness-target/blob/master/harness/decrypt_string.py) and cross your fingers

To push the app and native library to the device:

```bash
$ adb push app/build/output/apk/app-universal-debug.apk /data/local/tmp/target-app.apk
$ unzip app/build/outputs/apk/app-universal-debug.apk lib/x86/libstr-crypt.so
Archive:  app/build/outputs/apk/app-universal-debug.apk
  inflating: lib/x86/libstr-crypt.so
$ adb push lib/x86/libstr-crypt.so /data/local/tmp
lib/x86/libstr-crypt.so: 1 file pushed. 1.5 MB/s (5476 bytes in 0.004s)
```

To push harness to the device, 

```bash
cd harness
make && make install
```

**Note:** this pushes the x86 library to the device. If you really want to use another emulator image, replace `make install` with `adb push libs/<your emulator flavor>/harness /data/local/tmp`.

Now, run `harness` with the path to the target library as the first argument:

```bash
$ adb shell /data/local/tmp/harness /data/local/tmp/libstr-crypt.so
[*] Native Harness
 [+] Loading target: [ /data/local/tmp/libstr-crypt.so ]
 [+] Library Loaded!
 [+] Found JNI_OnLoad, good
 [+] Initializing JavaVM Instance
WARNING: linker: libdvm.so has text relocations. This is wasting memory and is a security risk. Please fix.
 [+] Initialization success (vm=0xb8e420a0, env=0xb8e420e0)
 [+] Calling JNI_OnLoad
 [+] Found decryptString function, good (0xb761f4f0)
 [+] Finding Cryptor class
 [+] Found Cryptor class: 0x1d2001d9
 [+] Found Cryptor.getInstance(): 0xb27bc270
 [+] Instantiated Cryptor class: 0x1d2001dd
 [+] Starting socket server on port 5001
```

To test that it's all working, in another terminal run:

```bash
$ ./decrypt_string.py
Sending encrypted string
Decrypted string: "Seek freedom and become captive of your desires. Seek discipline and find your liberty."
```

Finally, bask in your own greatness if you catch the reference, nerd.

## Conclusion

You should be able to take the harness code and modify the target function to run whatever function you want. This won't always work 100% reliably because programs are arbitrarily complicated and can do all kinds of weird shit.
