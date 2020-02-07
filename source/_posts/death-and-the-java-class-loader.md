title: "Death and the Java Class Loader"
tags:
  - smalivm
  - simplify
  - research
  - android
  - dalvik
comments: true
date: 2016-07-08 00:00
---

When [smalivm](https://calebfenton.github.io/2016/04/30/dalvik-virtual-execution-with-smalivm/) is virtually executing code, sometimes it needs to pass around Java Class objects. If it's a Java API class like `String` or `LinkedList`, that's no problem because smalivm is running in a JVM and has access to those classes already. But what if the class is of a type that's from the app it's trying to run? That class only exists in virtual execution imagination land, and if I don't want to rewrite everything and implement core JVM stuff myself, I need to dynamically create classes.

What this means is, when you pass smalivm an input DEX, it'll create a real life Java class which talks and walks just like the DEX class you give it, except it'll be inert and soulless, without any code. This way it can pass around the dry husk of a Java class, and if input DEX code wants to check the number of names of methods or do tricky reflection stuff, all those properties are there.
<!-- more -->

To accomplish this, I needed to create my own `ClassLoader`. I heartily recommend anyone to do this because I ended up learning a lot about Java in the process. Fear not comrades, I'm not actually going to write a Java tutorial (LOL), because I have to look at myself in the mirror in the morning and there's already CodeRanch. If you want to poke around the code though, [be my guest](https://github.com/CalebFenton/simplify/blob/master/smalivm/src/main/java/org/cf/smalivm/smali/SmaliClassLoader.java). Most of the ASM code I could find was written by overly clever academics and took my brain a while to parse. I wrote everything like an idiot, so it's probably easier to understand. A lot of the ASM heavy lifting happens in [org/cf/smalivm/smali/ClassBuilder](https://github.com/CalebFenton/simplify/blob/master/smalivm/src/main/java/org/cf/smalivm/smali/ClassBuilder.java).

I banged out most of the code during a 20 hour international flight. First, I wrestled with [ByteBuddy](http://bytebuddy.net/#/) for dynamic class generation. It's a great library, and the maintainer has a lot of documentation and even replied to my questions on Twitter, so he's cool, but it's ~`1_000` more complicated than I needed. It's the sort of library where stupid simple things are hard and confusing, but impossible stuff is possible and pretty clean. Eventually, I rewrote several hours of [complex, frustrated code](https://github.com/CalebFenton/simplify/blob/69944abc81bd3c3acee96381221eab95be5fb224/smalivm/src/main/java/org/cf/smalivm/smali/ClassBuilder.java) in 30 minutes just using [ASM](http://asm.ow2.org/).

However, during that time I was stuck on a plane. I had 4+ airline meals and whatever my wife wouldn't eat and constant access to beer. Also, I was coming back from visiting 2 of the 3 places in Vietnam with Dengue outbreaks (I'll spare you details, but pretty sure I got it) Combine that remaining more or less completely still in a cramped seat for 20 hours and you get this wicked awesome feeling of _actually dying_ IRL blended with ... constipation. My body acted like it was poisoned. Like with alcohol poisoning, it associated what I was doing (programming on Eclipse) with a horrible feeling of death and loneliness.  It was about a week before I could even look at [simplify](https://github.com/CalebFenton/simplify) code without feeling queasy, tired, and hopeless. I've since switched to IntelliJ. Lesson learned: fuck international travel.

![](/images/death-and-the-java-class-loader/finn-death.gif)

Everything was working great, then I ran into a snag. See, Android has a few classes that Java doesn't, because it's [totally and obviously a different thing guys](https://en.wikipedia.org/wiki/Oracle_America,_Inc._v._Google,_Inc.). Whenever an app used a `Class` defined in the Android framework but not in Java, I had to dynamically generate it. This was fine 99% of the time. **BUT.** Some class paths start with `java.` (e.g. `java.lang.DexCache`), and _thou shall not define classes which start with `java.`_. Here's the exception you get:

> Prohibited package name: java.net.DexCache also lol, eat dongs

It's a [security thing](http://stackoverflow.com/questions/3804442/why-java-lang-securityexception-prohibited-package-name-java-is-required).

I understand it's a security thing, but like any good security person, I figured I knew better and nothing bad would happen to me, so in my hubris I went to look if I could use any clever reflection tricks to _bypass the error_. To trace how this happens, start with your custom class loader. It'll call Java's `ClassLoader#defineClass()`:

```java
protected final Class<?> defineClass(String name, byte[] b, int off, int len,
                                     ProtectionDomain protectionDomain)
    throws ClassFormatError
{
    protectionDomain = preDefineClass(name, protectionDomain);
    String source = defineClassSourceLocation(protectionDomain);
    Class<?> c = defineClass1(name, b, off, len, protectionDomain, source);
    postDefineClass(c, protectionDomain);
    return c;
}
```

I know from the exception stack trace that there's a check in `preDefineClass`, so let's look there:

```java
private ProtectionDomain preDefineClass(String name,
                                        ProtectionDomain pd)
{
    if (!checkName(name))
        throw new NoClassDefFoundError("IllegalName: " + name);

    if ((name != null) && name.startsWith("java.")) {
        throw new SecurityException
            ("Prohibited package name: " +
             name.substring(0, name.lastIndexOf('.')));
    }
    if (pd == null) {
        pd = defaultDomain;
    }

    if (name != null) checkCerts(name, pd.getCodeSource());

    return pd;
}
```

Pff, this is easy. I'll just barf out some Java method to do all this myself, sans the inconvenient "java." check:

```java
private Class<?> dauntlessDefineClass(String name, byte[] b) throws Exception {
    // return defineClass(name, b, 0, b.length, null);

    Field f = ClassLoader.class.getDeclaredField("defaultDomain");
    f.setAccessible(true);
    ProtectionDomain protectionDomain = (ProtectionDomain) f.get(this);

    Method dcs = ClassLoader.class.getDeclaredMethod("defineClassSourceLocation",
                    new Class<?>[] { ProtectionDomain.class });
    dcs.setAccessible(true);
    String source = (String) dcs.invoke(this, protectionDomain);

    Method dc1 = ClassLoader.class.getDeclaredMethod("defineClass1", new Class<?>[] {
                    String.class, byte[].class, int.class, int.class, ProtectionDomain.class, String.class });
    dc1.setAccessible(true);
    Class<?> c = (Class<?>) dc1.invoke(this, name, b, 0, b.length, protectionDomain, source);

    Method pdc = ClassLoader.class.getDeclaredMethod("postDefineClass", new Class<?>[] {
                    Class.class, ProtectionDomain.class });
    pdc.setAccessible(true);
    pdc.invoke(this, c, protectionDomain);

    return c;
}
```

I'm most proud of the method name. Just [sounds cool](https://github.com/CalebFenton/simplify/blob/3dbfc719c88a7965e806f65efbd35d4cc495f173/smalivm/src/main/java/org/cf/smalivm/VirtualMachine.java#L192). But it didn't work. Turns out, the _real_ magic actually happens in *`defineClass1()`*. Good news! It's a native method! /s AFAIK, I can't do clever reflection to get around that. Since I need to call it to actually define the class and get my JVM `Class` object, the only solution seems to be drastically increase the complexity of running Simplify / smalivm such that I can hotswap in protected classes. I'll sit on that for now.

![](/images/death-and-the-java-class-loader/manly-tears.gif)