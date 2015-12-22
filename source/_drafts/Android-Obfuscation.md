title: Android Obfuscation
tags:
- obfuscation
- deobfuscation
- android
comments: true
---

This post covers common Android obfuscation techniques along with their evolution and adoption over time.

# Table Of Contents

## What is obfuscation?

To obfuscate is to hide. To say a program is obfuscated means the code has been made more difficult to understand without changing the behavior. Obfuscation can be used as an umberella term to refer to all types of code protection in general, including packing and anti-dissassembly tricks. However, it's usually used to refer to some type of post-compilation modification desgined to make the life of the reverser more "interesting".


## Brief history

In many ways the history of Android obfuscation has mirrored the PC scene from 20 years ago, but changes much faster. When I started in 2011, obfuscation was rare. I really only ever saw ProGuard, and that was uncommon because it wasn't yet bundled with the Android build environment. There were only a hand full of cases where developers understood how to hide things from reversers, but there were no tools to automate anything, so it had to be done by hand and was simple. One example was switching out a string for a character array. It's simple to do in Java, but looks much nastier as Smali.

A few years later, ProGuard is adopted into the Android build environment and its use became ubiquitous. Full-fledged commercial obfuscators like DexGuard hit the scene soon afterwards. The demand was likely spurred on by the fact that Android apps are much more easily reversed and cracked than PC applications. This is because they're written mostly in Java and protection schemes at the time were still quite simple.

Fast forward to 2015, and there are many commercial obfuscators and packers ready to mangle the ever living shit out of your code and make the life of a reverser quite "interesting" (read: tedious). The obfuscators weren't nearly as bad as they could be, probably because the people writing them weren't full time reversers. There's a good talk on the strength of obfuscators around this time by diff and jcase here: https://www.defcon.org/images/defcon-22/dc-22-presentations/Strazzere-Sawyer/DEFCON-22-Strazzere-and-Sawyer-Android-Hacker-Protection-Level-UPDATED.pdf


## Optimizations as obfuscation

Since these are done by ProGuard, they are very common. ProGuard is mainly an optimizer that obfuscates almost as an after thought. It's simple to setup and the obfuscations can make reversing a bit more tedious. It's been adopted into the official Androidroid build process for years now. Despite this, many companies release apps that haven't been ProGuarded! Oh well. Their lazyness is your boon.


### Identifier remapping

This is renaming classes and methods. I think it's called remapping because if you are a developer using this option in ProGuard, you can opt to get the remappings of all the names. This way you can see that `MyClass` was remapped to `a`, `MyFactoryClass` was remapped `b`, etc. You can also give it a mapping file to override the default alphabetical renaming.

You'll know it was ProGuarded, at least with the default options, if you see a lot of class and method names like `a`, `b`, `c`, ..., `aa`, `ab`, etc.

Before:
```
class DatabseHelper {
    public static boolean checkPassword(String username, String passwordHash) {
        // Some logic
    }
}
```

After:
```
class a {
    public static boolean a(String a, String b) {
        // Some logic
    }
}
```


### Debug symbol stripping

The java compiler normally includes debug symbols in the class bytecode. This includes variable names, line numbers, source file names, and other information useful for debugging. It's needed at run time to build call stacks for exceptions.

Removing these symbols can be annoying if you are a devloper and want useful stack traces from users' exceptions since they won't have line numbers. When combined with identifier remapping, the stack traces are even harder to decipher.

Look at all the useful information that's stripped away when symbols are removed.

Before:
```
.class public Lorg/cf/KwisatzHaderach;
.source "KwisatzHaderach.java"
.super Ljava/lang/Object;

.method public doCheck(IIII)Z
    .locals 3
    .param p0, "userId"    # I
    .param p1, "groupId"    # I
    .param p2, "permissionMask"    # I

    .line 1
    .local v0, "hasAccess":Z
    const/v v0, 0x0

    .line 2
    // Some logic

    return v0
.end method
```

After:
```
.class public Lorg/cf/KwisatzHaderach;
.super Ljava/lang/Object;

.method public doCheck(IIII)Z
    .locals 3

    const/v v0, 0x0

    // Some logic

    return v0
.end method
```


### Package hierarchy flattening

This is an agressive obfuscation option for ProGuard so you don't see it often. It requires devs to actually understand what a reverser sees and what makes their lives hard, which they don't. This option will try to squash all packages into the root package without breaking anything. It's also almost always combined with identifier remapping. So instead of looking at an app with a nice package structure and class names like this:
* com/apache/someLibrary
    * UtilityClass
    * SomeOtherClass
* org/cf
    * Main
    * Options
* org/cf/utils
    * HaxHelper

You'll end up looking at something like this, but much worse since there will be many more classes:
* a
* b
* c
* d
* e


## Literal encryption

Ok, we're done with the easy stuff. ProGuard doesn't do literal encryption. If you see it, you know it was either home rolled or a commercial obfuscator. A good way to understand how it should look is to understand how it's done. Take a moment and think how you'd encrypt all the string literals in an application without breaking it.

You done? Swell. There are lots of varations, but the basic idea is always the same. First, write a decryption method. Let's say it takes an encrypted string as input and outputs a decrypted string. Simple. Now, make it static and throw it into its own class so it can be called from anywhere. Then, encrypt all the strings. Finally, replace each unencrypted string with two things: the encrypted string and a call to the decryption method.

The actual encryption used doesn't really matter. The most common is some kind of cipher, usually xor. It's easy and, honestly, throwing up any kind of blockers at all is enough to deter 97% of noobs. If they splurge on more robust crypto, the key is probably static and floating around hard-coded somewhere. If they do something wild like get the key from a server, the singing cert

One varation is to have multiple decryption methods, e.g. one for each class. Each method could be a little different. I don't know why people do this, but I see it from time to time. If you're really dumb, you might reverse each method independently instead of coming up with a more general decryption solution, which I'll subtly reference here and there in this post, and will write about properly in the future. Another variation is to use one or more integers as an argument to the decryption method. This is possible since encrypted strings are known ahead of time, and can be included in the decryption method such that they can be indexed by an integer. This has a little added benefit of globally uniquing strings.

Similar techniques can be applied to numbers and arrays.


## Control flow redirection
at the beginning of a method, go to the end, and back again
throws off some static analysis and code similarity tools
also difficult to read
whole method one big switch statement


## White noise

Integer.valueOf
x = 5; 1 + 2 + 3 * 4 / 5 % 8; return x;
show some examples


## Reflection

used in combination
from 0bad


## Dynamic loading

strings in separate file
encrypted or downloaded dex or apk
foundation of packing


## Anti-disassembly

breaking tools
bad dead code
jumping into arrays
http://dexlabs.org/blog/bytecode-obfuscation - jump into arrays, other good stuff
http://archive.hack.lu/2013/AbusingDalvikBeyondRecognition.pdf

## Virtual machines

used in x86 more frequently
seen in malware, simpletemai


## Native code

semi-obfuscating


## References

include link from dropbox pdf

http://ieeexplore.ieee.org/xpl/login.jsp?tp=&arnumber=1621528&url=http%3A%2F%2Fieeexplore.ieee.org%2Fiel5%2F5658%2F33968%2F01621528.pdf%3Farnumber%3D1621528
