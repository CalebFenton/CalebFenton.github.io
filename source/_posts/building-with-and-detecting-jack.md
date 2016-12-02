# Building with and Detecting Jack

Recently, I needed to write a bunch of Smali to use in tests for [Simplify](https://github.com/CalebFenton/simplify). Writing Smali is pretty easy, but it's tedious, and I needed to do tricky stuff with nested classes. As I mentioned in a [previous post](https://calebfenton.github.io/2016/07/31/understanding_dalvik_static_fields_1_of_2/) it's possible to setup an alias to quickly compile Java code to Dalvik. This way, you can prototype Java quickly and compile it without worrying about Smali syntax. In this post, I want to show how to use a new tool called [`jack`](https://source.android.com/source/jack.html) which takes the place of `dx`.
<!-- more -->

## Building with Jack
The previous decompiler is `dx` which operates on Java _.class_ files. Jack works directly on Java source code. It looks like dx isn't supported anymore and it doesn't work with newer Java versions. In fact, the reason I had to learn about Jack at all is because my old `java2smali` alias was giving me the following error:

```bash
$ dx --dex AndroidException.class

PARSE ERROR:
unsupported class file version 52.0
...while parsing AndroidException.class
1 error; aborting
```

Looks like `dx` doesn't support Java 8 classes. After a few milliseconds panic, I calmed down and started looking through the `build-tools` directory of the most recent Android platform I had installed:

```bash
$ echo $ANDROID_BUILD_TOOLS
/Users/caleb/android-sdk/build-tools/25.0.0
```

This is where I found `jack.jar`. Running it gives this nice, long, interesting help message:

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

I was curious if it was possible to fingerprint Jack built _.dex_ files. If so, I wanted to add the fingerprints to the database in [APKiD](https://github.com/rednaga/APKiD). To find how the files are different, I built two _.dex_ files from the same Java but using different tools. In this case, `dx` and `jack.jar`.

Here's a small _.dex_ file created with `dx`:

![](/images/building-with-and-detecting-jack/dx-dex.png)

Here's the same Java converted with `jack.jar`:

![](/images/building-with-and-detecting-jack/jack-dex.png)

See that cute little `emitter: jack-4.12`? Looks like Jack intentionally watermarks files it creates. It might be able to turn it off with a command line parameter, but I haven't looked. Here are the rules I added to APKiD to detect Jack: [https://github.com/rednaga/APKiD/commit/ccca5ed519b7b2551a3205686be364c26020f1cd#diff-1731ed362177d8429d827a2b6ef3786bR131](https://github.com/rednaga/APKiD/commit/ccca5ed519b7b2551a3205686be364c26020f1cd#diff-1731ed362177d8429d827a2b6ef3786bR131).