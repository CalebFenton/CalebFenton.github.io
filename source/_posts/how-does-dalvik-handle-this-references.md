
title: How does Dalvik handle this references?
tags:
  - research
  - android
  - dalvik
comments: true
date: 2016-02-21 12:44:39
---

## The `this` Reference
For every instance (virtual, non-static) method in Dalvik, the first parameter is a reference to itself, or, in Java, the `this` reference. I wanted to know if it was legal to reassign the register value.

Just so I'm sure you know what I'm talking about, here's a simple Java class with an instance method called `instanceMethod`:

```java
public class Instance {
    private int number = 5;

    public int instanceMethod() {
        return this.number;
    }
}
```
<!-- more -->

The above smali gets turned into this (you can safely ignore `<init>()V`):

```smali
.class public LInstance;
.super Ljava/lang/Object;

.field private number:I

.method public constructor <init>()V
    .locals 1
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V
    const/4 v0, 0x5
    iput v0, p0, LInstance;->number:I
    return-void
.end method

.method public instanceMethod()I
    .locals 1

    # p0 is the 'this' reference
    iget v0, p0, LInstance;->number:I

    return v0
.end method
```

Do your decompilations look different? It may be because mine was generated using `baksmali --use-locals` which separates the registers into registers used within the method body (locals) and those passed as parameters. Local registers are named `v0`, `v1`, `v2`, and so on and the parameters are named `p0`, `p1`, etc.

The default behavior is to name all registers based on how they're actually laid out by Dalvik: `r0`, `r1`, `r2` and so on, regardless of if they're local or parameters. To clarify, a method like this:
```smali
public example(JI)V
    .registers 3
```

Has a register layout like this:
* `r0`, `r1`, `r2` - local registers
* `r3` - `this` register (p0 with `--use-locals`)
* `r4` & `r5` - `J` parameter (wide types use two registers)
* `r6` - `I` parameter

## The Question

I wondered if `p0` was somehow special and if it was possible to rewrite it. One of the optimizers I'm working on needs to rewrite Smali and it works best if it knows all of the available registers at a certain point in code. A register is "available" if it's not used for the rest of the execution. If you've ever written a tool to automatically modify Smali, you have probably run into this problem.

**Spoiler warning:** It is _not_ special and it _is_ possible to reassign `p0`!

Here's the code I used to test:

```smali
.class public LHello;
.super Ljava/lang/Object;
.source "Hello.java"

.method public constructor <init>()V
    .locals 0
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V
    return-void
.end method

.method public static main([Ljava/lang/String;)V
    .locals 2
    .param p0, "argv"    # [Ljava/lang/String;
    sget-object v0, Ljava/lang/System;->out:Ljava/io/PrintStream;
    new-instance v1, LHello;
    invoke-direct {v1}, LHello;-><init>()V
    invoke-virtual {v1}, LHello;->instance()I
    move-result v1
    invoke-virtual {v0, v1}, Ljava/io/PrintStream;->println(I)V
    return-void
.end method

.method public instance()I
    .locals 1

    # rewrite p0 with 0x5, cross fingers, hope it works
    const/4 p0, 0x5
    return p0
.end method
```

And then to compile and run it:

```bash
$ smali hello.smali -o classes.dex && zip Hello.zip classes.dex && adb push Hello.zip /data/local && adb shell dalvikvm -cp /data/local/Hello.zip Hello
  adding: classes.dex (deflated 45%)
115 KB/s (619 bytes in 0.005s)
5
```

The test code outputs the expected `5` with no errors or warnings. It makes sense that a register should be able to hold a reference to anything, but the only way to be absolutely sure (without closely examining the source) is to test it.
