
title: How does Dalvik handle null?
tags:
  - research
  - android
  - dalvik
comments: true
date: 2016-02-16 19:44:39
---

## The Problem
Dalvik doesn't have a proper null type. A null is [represented by a `0`](https://github.com/android/platform_dalvik/blob/master/dx/src/com/android/dx/rop/cst/CstKnownNull.java#L92). Consider this example Smali code:
`const/4 v0, 0x0`

It could actually represent a few of different types:
* `int v0 = 0;`
* `boolean v0 = false;`
* `byte v0 = 0x0;`
* `short v0 = 0;`
* And even: `v0 = null;`
<!-- more -->

In case you were wondering about how `char` is handled, `char c = 'a'` it looks like this:
`const/16 v0, 0x61`

I wanted to know when Dalvik coaxed `0` values into `null` references for my work on [Simplify](https://github.com/CalebFenton/simplify). I tried searching and only found bits and pieces, and, of course, a bunch of source code. The first page I found that looked promising was [http://forensics.spreitzenbarth.de/2012/08/27/comparison-of-dalvik-and-java-bytecode/
](http://forensics.spreitzenbarth.de/2012/08/27/comparison-of-dalvik-and-java-bytecode/
) but all it said about nulls was:
> Dalvik bytecode does not have a specific null type. Instead, Dalvik uses a 0 value constant. So, the ambiguous implication of constant 0 should be distinguished properly.

## The Experiment
I dug through the source code a little and felt like I only partially understood when it happened. To be sure, and to understand more deeply, and mostly because I like to do things the cheap, easy, ghetto way, I decided to write some Java, convert it to Smali, and execute it to see what happens!

Consider this bit of code which handles `null` and `0` back to back:
```java
public static void addNullAnd0ToList() {
    List<Integer> wtf = new LinkedList<Integer>();
    wtf.add(null);
    wtf.add(0);
    System.out.println(wtf); // [null, 0]
}
```

This is the resulting Smali (with a `main` method that I added because I'm nice and want you to be able to easily execute this yourself):
```smali
.class public LHelloWorld;
.super Ljava/lang/Object;

.method public static varargs main([Ljava/lang/String;)V
    .locals 0
    .prologue
    invoke-static {}, LHelloWorld;->addNullAnd0ToList()V
    return-void
.end method

.method public static addNullAnd0ToList()V
    .locals 4
    .prologue
    new-instance v0, Ljava/util/LinkedList;
    invoke-direct {v0}, Ljava/util/LinkedList;-><init>()V

    .local v0, "wtf":Ljava/util/List;, "Ljava/util/List<Ljava/lang/Integer;>;"
    const/4 v1, 0x0
    invoke-interface {v0, v1}, Ljava/util/List;->add(Ljava/lang/Object;)Z

    const/4 v1, 0x0
    invoke-static {v1}, Ljava/lang/Integer;->valueOf(I)Ljava/lang/Integer;
    move-result-object v1
    invoke-interface {v0, v1}, Ljava/util/List;->add(Ljava/lang/Object;)Z

    sget-object v1, Ljava/lang/System;->out:Ljava/io/PrintStream;
    invoke-virtual {v1, v0}, Ljava/io/PrintStream;->println(Ljava/lang/Object;)V

    return-void
.end method
```

These two lines are responsible for adding the `null` to `wtf:Ljava/lang/List;`:
```smali
const/4 v1, 0x0
invoke-interface {v0, v1}, Ljava/util/List;->add(Ljava/lang/Object;)Z
```

My first guess was Dalvik sees that `v1` contains an integer but is used as a `Ljava/lang/Object;` type argument. Does it have to be an integer? Does it work with other numbers? What if `v1` was a `short`? I added a `check-cast` to force `v1` into `short`:

```smali
const/4 v1, 0x0
check-cast v1, S
invoke-interface {v0, v1}, Ljava/util/List;->add(Ljava/lang/Object;)Z
```

Then compiled an ran everything:
`smali hello.smali -o classes.dex && zip Hello.zip classes.dex && adb push Hello.zip /data/local && adb shell dalvikvm -cp /data/local/Hello.zip HelloWorld`

It failed:
```
DexOpt: --- BEGIN 'Hello.zip' (bootstrap=0) ---
DexOpt: load 4ms, verify 4ms, opt 0ms
DexOpt: --- END 'Hello.zip' (success) ---
DEX prep '/data/local/Hello.zip': unzip in 0ms, rewrite 58ms
VFY: S is not instance of Ljava/lang/Object;
VFY: bad arg 1 (into Ljava/lang/Object;)
VFY:  rejecting call to Ljava/util/List;.add (Ljava/lang/Object;)Z
VFY:  rejecting opcode 0x72 at 0x0008
VFY:  rejected LHelloWorld;.addNullAnd0ToList ()V
Verifier rejected class LHelloWorld;
```

The key part of this error is the `S is not instance of Ljava/lang/Object;`. Ok, that's fair. There must be a difference between registers with and without explicit type casting. But does it work with integers? I tried with `check-cast v1, I` and got about the same error. The code didn't get past the verifier, so it knew at runtime it was wrong. To use a `short` without a `check-cast` I just added a `getShort()S` method. I didn't think it would work because in both the method call and with `check-cast`, explicit type information is available.

```smali
invoke-static {}, LHelloWorld;->getShort()S
move-result v1
```

```smali
.method public static getShort()S
    .locals 1
    const/4 v0, 0x0
    return v0
.end method
```

And I was right; it fails:
```
VFY: register1 v1 type 10, wanted ref
VFY: bad arg 1 (into Ljava/lang/Object;)
```

This is getting silly and I'm starting to think I should maybe just audit the source to fully understand. So I spend another 10 - 15 minutes poking around before giving up. I'll just derrive the behavior experimentally _hashtag yolo_.

For the sake of completeness, I also try with a `getInt()I`:
```
invoke-static {}, LHelloWorld;->getInt()I
move-result v1
```

```
.method public static getInt()I
    .locals 1
    const/4 v0, 0x0
    return v0
.end method
```

Another failure:
```
VFY: register1 v1 type 12, wanted ref
```

Dalvik can see through my cheap tricks. What if I try a wide value like with `const-wide`? There's no _explicit_ type... Slight change to the code because `long`s are fat and take up two registers. I had to move the register to `v2`.

```
const-wide v2, 0x0L
```

NOPE:
```
VFY: register1 v2 type 13, wanted ref
```

## Conclusion

Eventually, I found that only two things work for a `null`:
1. `const/4 v1, 0x0`
2. `const/16 v1, 0x0`

And these are considered `null` _only_ if there's no explicit type information available between assignment and use. Now I can take these delicious, esoteric trivialities and apply them towards creating failing tests. And I can't help but simultaneously get excited by the prospect of failing tests and wonder what kind of life choices led to this.
