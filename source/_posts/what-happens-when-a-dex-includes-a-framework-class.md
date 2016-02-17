title: What happens when a DEX includes a framework class?
tags:
  - research
  - android
  - dalvik
comments: true
date: 2015-12-21 20:13:40
---

## Why
While working on a new class loading system for [SmaliVM](https://github.com/CalebFenton/simplify/tree/master/smalivm), I needed to understand exactly how DalvikVM handles the case of a DEX file including a system / framework class such as `Ljava/lang/Object;`. I'd originally assumed, naively, in retrospect, that class files in a DEX file should take precedence. Thinking about this for a half second, I have no idea what the hell I was thinking. That would be _stupid_.
<!-- more -->

If Dalvik let apps redefine framework classes, it has huge security implications. Sure, each app runs it its own zygote-spawned sandbox, but what if somewhere, somehow, my malicious app's DEX file was loaded by an app with system or root access? I could just backdoor `Ljava/lang/Object;`. Even if that's not possible, I'm sure I could think of something nefarious if you gave me the ability to backdoor any class.

Well, derp, so now I have to rewrite part of [Simplify](https://github.com/CalebFenton/simplify) and (hopefully) fix some tests. I might as well know exactly how it fails and document it for other researchers, right?

## How
First, I created two small Smali files.

smali/**hello.smali**:

``` smali
.class public LHelloWorld;
.super Ljava/lang/Object;

.method public static main([Ljava/lang/String;)V
    .locals 2

    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
    const-string v1, "hello,world!"
    invoke-virtual {v0, v1}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    return-void
.end method
```

The purpose of this is just to provide a `main(String[])` method and to have `Object` as a `super`.

smali/**object.smali**:

``` smali
.class Ljava/lang/Object;

.method public static <clinit>()V
    .locals 2

    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
    const-string v1, "crazyballs"
    invoke-virtual {v0, v1}, Ljava/io/PrintStream;->println(Ljava/lang/String;)V

    return-void
.end method
```

This is the real test. If I can overwrite framework classes, I should get a lot of errors, duh, but not before `<clinit>` prints out something witty.

After that, it was just packing it up and shoving it onto an emulator:

``` bash
$ smali smali -o classes.dex
$ zip hello.zip classes.dex
  adding: classes.dex (deflated 47%)
$ adb push hello.zip /data/local
```

I'll also wanted to see the error explosions in the logs. You'd be surprised how many people have an app crash or something and don't bother looking at the logs. `monitor` is your friend. It usually has bad news, and confuses Eclipse and IntelliJ if it's running, but at least it's honest.

``` bash
$ monitor &disown
```

Finally, just invoke `dalvikvm` with our ZIP as the classpath:

``` bash
$ adb shell
root@android:/ # cd /data/local
root@android:/data/local # dalvikvm -cp hello.zip HelloWorld
hello,world!
root@android:/data/local #
```

No `"crazyballs"`, so I guess my hunch was right. I wonder what the error looked like?

```
12-21 11:07:22.035: D/dalvikvm(1065): DexOpt: --- BEGIN 'hello.zip' (bootstrap=0) ---
12-21 11:07:22.095: D/dalvikvm(1066): DexOpt: 'Ljava/lang/Object;' has an earlier definition; blocking out
12-21 11:07:22.095: D/dalvikvm(1066): DexOpt: not verifying/optimizing 'Ljava/lang/Object;': multiple definitions
12-21 11:07:22.115: D/dalvikvm(1066): DexOpt: load 14ms, verify+opt 11ms, 83668 bytes
12-21 11:07:22.115: D/dalvikvm(1065): DexOpt: --- END 'hello.zip' (success) ---
12-21 11:07:22.115: D/dalvikvm(1065): DEX prep 'hello.zip': unzip in 2ms, rewrite 75ms
```

If you read between the lines, the actual error message is "has an earlier definition; blocking out (idiot)".
