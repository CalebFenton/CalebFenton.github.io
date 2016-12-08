title: "Building with and Detecting Jack"
tags:
  - android
comments: true
date: 2016-12-01 00:13:37
---

Recently, I needed to write a bunch of Smali code to use in tests for [Simplify](https://github.com/CalebFenton/simplify). While, Smali syntax is simple and fairly easy to write, it's also tedious and I needed to do some tricky, uncommon stuff. I wasn't even sure how to do it in Smali. Luckily, it's pretty easy to write Java and convert it to Smali. I've talked about how to make a small alias to do this and go over some other use cases in a [previous post](https://calebfenton.github.io/2016/07/31/understanding_dalvik_static_fields_1_of_2/). Writing Java and converting to Smali makes it easy to quickly prototype lots of Smali code without worrying about Smali syntax or conventions. In this post, I want to show how to use a new Android compiler called [`jack`](https://source.android.com/source/jack.html) which takes the place of `dx` and you'll need to know how to use if you want to continue converting Java to Smali.
<!-- more -->

## Building with Jack

The original Android compiler is `dx` and it works by translating Java _.class_ files to Dalvik executables (_.dex_). Jack, however, compiles Java source code, so you don't need to invoke `javac` at all.

I originally had to learn about Jack because it looks like `dx` doesn't support newer Java _.class_ versions. I tried converting a Java 8 compiled class and `dx` gave me the following error:

```bash
$ dx --dex AndroidException.class

PARSE ERROR:
unsupported class file version 52.0
...while parsing AndroidException.class
1 error; aborting
```

This was a problem. I was trying to fix a bug which exposed another bug which exposed 4 or 5 things I could be doing much cleaner which in turn led to even more stuff I wanted to fix _first_ before fixing the original bugs. It's a bit like this:

![](http://i.imgur.com/t0XHtgJ.gif)

By the time I saw the `dx` error, I had about blown my stack. After a few milliseconds panic, I calmed down and started looking through the `build-tools` directory of the most recent Android platform I had installed. I knew that Android must be able to convert Java 8 classes to _.dex_ because people are making Android apps using it.

```bash
$ echo $ANDROID_BUILD_TOOLS
/Users/caleb/android-sdk/build-tools/25.0.0
```

Lo and behold, there's this `jack.jar`. I ran it just to see what it did and it gave this nice, long, helpful help message:

```bash
$ java -jar $ANDROID_BUILD_TOOLS/jack.jar --help
Usage: <options> <source files>

Options:
 @<FILE>                                          : read command line from file
 --config-jarjar <FILE>                           : use jarjar rules files (default: none)
 --config-proguard <FILE>                         : use a proguard flags file (default: none)
                                                    (repeatable)
 --help                                           : display help
 --help-properties                                : display properties list
 --import <FILE>                                  : import the given file into the output
                                                    (repeatable)
 --import-meta <DIRECTORY>                        : import the given directory into the output as
                                                    meta-files (repeatable)
 --import-resource <DIRECTORY>                    : import the given directory into the output as
                                                    resource files (repeatable)
 --incremental-folder <DIRECTORY>                 : directory used for incremental data
 --list-plugins                                   : display all available plugins
 --multi-dex [NONE | NATIVE | LEGACY]             : whether to split code into multiple dex files
                                                    (default: none)
 --output-dex <DIRECTORY>                         : output dex files and resources to the directory
 --output-jack <FILE>                             : output jack library file
 --plugin <NAME>[,<NAME>...]                      : jack plugin names
 --pluginpath <PATH>                              : jack plugin classpath
 --processor <NAME>[,<NAME>...]                   : annotation processor class names
 --processorpath <PATH>                           : annotation processor classpath
 --verbose [ERROR | WARNING | INFO | DEBUG |      : set verbosity (default: warning)
 TRACE]
 --version                                        : display version
 -A <option>=<value>                              : set option for annotation processors
                                                    (repeatable)
 -D <property>=<value>                            : set value for the given property (repeatable)
 -cp (--classpath) <PATH>                         : set classpath
 -g                                               : emit debug infos
```

The main difference between `jack.jar` and `dx` is that, since Jack operates on Java source, it needs access to some Java runtime files. I assumed `rt.jar` would be enough. Here's an example of how to use it with a file called `Hello.java`:

```bash
mkdir temp_dex
java -jar $ANDROID_BUILD_TOOLS/jack.jar -cp `/usr/libexec/java_home`/jre/lib/rt.jar --output-dex temp_dex AndroidException.java
baksmali d -o temp_smali temp_dex/classes.dex
```

This is tweaked to work on a Mac, but it should be easy to translate to Linux or Windows.

## Detecting Jack Created Files

I was curious if it was possible to fingerprint Jack-built _.dex_ files. There's a lot of useful stuff you can do with [compiler fingerprinting](http://rednaga.io/2016/07/30/apkid_and_android_compiler_fingerprinting/) such as detecting malware. I wanted to add the fingerprints to the database in [APKiD](https://github.com/rednaga/APKiD). To find how the files are different, I built two _.dex_ files from the same Java but using different tools. In this case, `dx` and `jack.jar`.

Here's a small _.dex_ file created with `dx`:

![dx generated dex](/images/building-with-and-detecting-jack/dx-dex.png)

Here's the same Java converted with `jack.jar`:

![jack generated dex](/images/building-with-and-detecting-jack/jack-dex.png)

See that cute little `emitter: jack-4.12`? Looks like Jack intentionally watermarks files it creates. It might be able to turn it off with a command line parameter, but I haven't looked. Here are the rules I added to APKiD to detect Jack: [https://github.com/rednaga/APKiD/commit/ccca5ed519b7b2551a3205686be364c26020f1cd#diff-1731ed362177d8429d827a2b6ef3786bR131](https://github.com/rednaga/APKiD/commit/ccca5ed519b7b2551a3205686be364c26020f1cd#diff-1731ed362177d8429d827a2b6ef3786bR131).


### Update 12-06-2016 - Improved Jack Detection

Shortly after announcing this blog on Twitter, [@iBotPeaches](https://twitter.com/iBotPeaches) (Apktool developer) was kind enough to [point out](https://twitter.com/iBotPeaches/status/804772366266015744) another distinguishing characteristic of Jack-generated DEX files which is described here: [https://github.com/iBotPeaches/Apktool/issues/1354](https://github.com/iBotPeaches/Apktool/issues/1354).

The new Jack compiler changes the names of [compiler generated `access$000` methods](http://vanillajava.blogspot.com/2011/07/java-secret-generated-methods.html) to names like `-set0()`, `-get0()`, and `-wrap0()`. To build some _.dex_ files for testing, here's some Java code:

```java
public class OuterClass {
  private static final int myInt;

  private static void baz() {
  }

  static class InnerClass {
    private void foo() {
      // -set0()
      myInt = 5;
    }

    private void bar() {
      // -get0()
      System.out.println(myInt);
    }

    private void qux() {
      // -wrap0()
      baz();
    }
  }
}
```

I saved this as `OuterClass.java` and compiled it explicitly with Java 1.7 because 1.8 isn't supported by `dx`:

```bash
`/usr/libexec/java_home -v 1.7`/bin/javac OuterClass.java
dx --dex --output=dx.dex *.class
```

Then, I used `javap` to get the names of the `javac` generated method names:

```
$ javap OuterClass.class
Compiled from "OuterClass.java"
public class OuterClass {
  public OuterClass();
  static int access$002(int);
  static int access$000();
  static void access$100();
}
```

Yup, it's all `access$00\d` stuff, so Jack must be the one changing these. No idea why it does this apart from making it a bit more clear what their function is, i.e. it's easier to guess the behavior of `-set0(int)` than `access$002(int)`.

To get a Jack compiled version of `OuterClass`:

```bash
mkdir jack
java -jar $ANDROID_BUILD_TOOLS/jack.jar -cp `/usr/libexec/java_home -v 1.7`/jre/lib/rt.jar OuterClass.java --output-dex jack
```

Now, to examine the differences in method names by looking at the strings:

![dx and jack strings](/images/building-with-and-detecting-jack/dx_and_jack_juxtaposed.png)

Ok, seems obvious enough. I could look for these two sets of strings to figure out the compiler. But what happens if you use `dexmerge` to combine the `dx` and `jack.jar` produced _.dex_ files?

Here's the alias I use for `dexmerge`:

```bash
$ alias dexmerge
dexmerge='java -cp $ANDROID_BUILD_TOOLS/lib/dx.jar com.android.dx.merge.DexMerger'

$ echo $ANDROID_BUILD_TOOLS
/Users/caleb/android-sdk/build-tools/25.0.0
```

In case you're curious, here's the usage:

```
$ dexmerge
Usage: DexMerger <out.dex> <a.dex> <b.dex> ...

If a class is defined in several dex, the class found in the first dex will be used.
```

My guess was that this would merge all the strings but the only the code in the first _.dex_ would be kept, leaving several strings unreferenced. Unreferenced strings might actually be an interesting heuristic for finding [weird](http://i.imgur.com/V7Htnoe.gif) _.dex_ files, but it's not something you could do without some disassembly.

```bash
$ dexmerge merge.dex dx.dex jack/classes.dex
Merged dex #1 (2 defs/1.3KiB)
Merged dex #2 (2 defs/1.3KiB)
Result is 2 defs/2.5KiB. Took 0.0s
```

Lo, and behold, the strings from each are retained:

![dexmerge strings](/images/building-with-and-detecting-jack/dexmerge.png)

This _.dex_ has an interesting history. If you know what to look for, you could tell quite a bit about how it was made which may help you infer how technically sophisticated the creator was and what tools and environment they were using. You'd know straight away that it's the result of `dexmerge` because of the [map type ordering](https://hitcon.org/2016/CMT/slide/day1-r0-e-1.pdf) (search `ABNORMAL_TYPE_ORDER` (note to self: number slides in the future)). You would also know parts of the file were created with `dx` or dexlib 2.x and parts were created with `jack.jar`.