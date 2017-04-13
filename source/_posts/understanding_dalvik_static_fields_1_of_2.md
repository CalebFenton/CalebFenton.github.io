title: Understanding Dalvik Static Fields part 1 of 2
tags:
  - research
  - android
  - dalvik
comments: true
date: 2016-07-31
---

This story starts with [someone](https://github.com/teDDyGH) reporting a very well written and concise [issue](https://github.com/CalebFenton/simplify/issues/50) for Simplify. After digging into it, I found a problem with how smalivm was handling static field initialization. In case you didn't know, you can initialize a static field in smali like this:

```smali
.field private static someInt:I = 5
```

I'd seen that smali supported this format years ago, and included it in my [Smali syntax definitions for Sublime](https://github.com/ShaneWilton/sublime-smali), but I couldn't ever produce a DEX which used this. Whenever I had a simple, primitive static field, `dx` would generate bytecode which initialized the field in the class initializer `<clinit>`.

Ok, so now I needed to support this in smalivm which means I had to figure out _exactly_ how everything worked, what was valid, what was invalid, and how each type (float, long, int, ...) looks. Yay!

<!-- more -->

Long ago, when I tried to create a DEX which had these "inline static field literals", the reason I failed may have been because I was either using an older version of `dx` or I was invoking it weirdly (looking at you `--no-optimize`). If some versions just didn't use inline literals, it could be an interesting signature for compiler fingerprinting for [APKiD](https://github.com/rednaga/APKiD).

When thinking about this problem, I realized I didn't think very carefully about the values of uninitialized fields. Originally, I was assuming an `UnknownValue` for all fields until they were initialized. However, I know Java treats Objects as `null` and primitives as something sensible like `0` for numerics and `false` for boolean. I felt absolutely confident that Dalvik worked the same way, so of course I setup a way of testing it to be absolutely super confident + 1. So, I wrote some Java:

```java
class InitTests {
  private static int someInt;
  private static char someChar;
  public static void main(String[] args) {
    System.out.println("someInt = " + someInt);
    System.out.println("someChar = " + someChar);
    System.out.println("is it eql? " + (someChar == '\0'));
  }
}
```

This will test the default value of `int` and `char`. I bet you didn't think about `char` when I mentioned default values of primitives earlier! Oh no, you were probably smugly thinking "of course an `int` is 0, that just makes sense, duh!" but what about `char`, huh? I figured it'd probably be a null character, but I so infrequently use those in Javaland that I wasn't even sure `'\0'` would work. Turns out, it does.

So I have all this Java. How am I going to run this on a Dalivk VM? You might be thinking, "Oh! I know this one! I'll make an Android project, add this code as a class, wire it up to get executed when the main activity loads, and throw it on an emulator!" If that's what you thought, give yourself an "**F**" because **F**uck that. Way too slow. Enter `java2smali`.

```bash
# Need javac, baksmali and dx on your path, bro
function func_java2smali() {
  if [ -z "${1}" ]; then
    echo "usage: java2smali <java file>"
    return
  fi

  filename=$(basename $1 .java)
  outDir=$(shasum $1 | awk '{print $1}')

  # Generate debug info in case we keep the dex
  javac -g -cp $ANDROID_PLATFORM/android.jar $1
  dx --dex --no-strict --no-optimize --output=$filename.dex $filename.class
  baksmali --sequential-labels --use-locals $filename.dex -o $outDir

  cp -R $outDir/**/*.smali .

  rm -r $outDir
  rm $filename.class
  rm $filename.dex
}
alias java2smali=func_java2smali
```

This bad bitch will convert your easy to read and write Java code into Smali. It has some limitations. Namely, it's written in Bash, so a small percentage of you may go mad if you look at it too long. It also doesn't work super good with multiple files or inner classes.

Now you can generate Smali from the Java, but you still need to execute it, right? Enter `runsmali`:

```bash
function func_runsmali() {
  if [ -z "${1}" ] || [ -z "${2}" ]; then
    echo "usage: runsmali <smali file or dir> <class> [adb device]"
    return
  fi

  smali $1 -o classes.dex
  zip runsmali.zip classes.dex
  rm classes.dex

  # SUCH ROBUST. MUCH RELIABLE.
  TEMP_ADB_PATH="/data/local"
  OUTPUT=`adb shell ls $TEMP_ADB_PATH`
  if [[ $OUTPUT == *"Permission denied"* ]]
  then
    TEMP_ADB_PATH="/data/local/tmp"
  fi

  if [ -z "${3}" ]; then
    adb push runsmali.zip $TEMP_ADB_PATH
  else
    adb push -s $3 runsmali.zip $TEMP_ADB_PATH
  fi
  rm runsmali.zip

  echo "EXECUTING: $2"
  adb shell dalvikvm -cp $TEMP_ADB_PATH/runsmali.zip $2
}
alias runsmali=func_runsmali
```

Ideally, this should be in Python and installable using `pip`. It's a rainy day project, and since it doesn't rain in California, it might be a while before I get to it. Also, about Python packages, friends, let me tell you that Python packaging is a _DARK ART_. You want to know the best practices? Fuck you. That's what they are. It's a mess. Ruby Gems are _much_ easier, but Ruby isn't installed on everyone's machine like Python is.

![](/images/understanding-dalvik-static-fields/black_magic.png)

> Disclaimer: Coding in Python is a joy, and I don't dislike the language at all. There's just some confusing shit if you're new and trying to learn, like Python 2 vs Python 3 and all the different package distribution tools.

Anyway, just fire up an emulator and use `runsmali` to take care of invoking the dalvikvm and get the output:

```
$ runsmali InitTests.smali InitTests
  adding: classes.dex (deflated 44%)
[100%] /data/local/runsmali.zip
EXECUTING: InitTests
someInt = 0
someChar =
is it eql? true
```

I wanted to know how to initialize all of the different types, so I had to go digging through the syntax to know everything that was valid. I started [here](https://github.com/JesusFreke/smali/blob/master/smali/src/main/antlr/smaliParser.g#L481) where `FIELD_DIRECTIVE` is defined. This led me to some literal parsing code [here](https://github.com/JesusFreke/smali/blob/master/smali/src/main/antlr/smaliTreeWalker.g#L267). Ultimately, I found [this](https://github.com/JesusFreke/smali/blob/master/smali/src/main/jflex/smaliLexer.jflex#L288) which told me how type signifiers are defined. I also stumbled across [this](https://github.com/JesusFreke/smali/blob/master/smali/src/main/jflex/smaliLexer.jflex#L203) which showed me `float`s and `double`s can be `NaN` in addition to numeric literals. That leaves us with these all as valid static field literals:

```smali
.field static myInt:I = -4
.field static myShort:S = 0xDEADS
.field static myBoolean:Z = false
.field static myChar:C = '\n'
.field static myLong:J = 1000000000l
.field static myOtherLong:J = 0x42424242L
.field static myFloat:F = NaN
.field static myFloat2:F = NaNf
.field static myOtherFloat:F = 3.14159265357
.field static myOtherFloat2:F = 3.14159265357f
.field static myDouble:D = 10000000.9d
.field static myObject:Ljava/lang/Object; = null
.field static myString:Ljava/lang/String; = "Neuro from the nerves, the \"silver\" paths."
.field static myByte:B = 0x5t
```

In the next post on this topic, I'll talk about inheritance and valid ways of referencing fields.