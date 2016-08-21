title: Understanding Dalvik Static Fields part 2 of 2
tags:
  - research
  - android
  - dalvik
comments: true
date: 2016-08-21
---

In the [first part](2016/07/31/understanding_dalvik_static_fields_1_of_2/) of this series on Dalvik class fields, I wrote about how Dalvik handles static field literals. This article is focused on how field inheritance works and exploring all the different but equally valid ways of referencing fields at the bytecode level.

If you are familiar with Java, you probably already understand how Java field inheritance looks and behaves at the source code level, but btyecode is less strict and potentially more ambiguous (at least to humans) than source. JVM languages like Scala and Groovy compile to the same bytecode as Java, but both have very different source code restrictions.
<!-- more -->

Since I'm doing this for [Simplify](https://github.com/CalebFenton/simplify), the first question is what are _all_ the different ways static field references can look so I can test they're handled correctly. Here's an example field reference:

```smali
Lorg/cf/ChildClass;->myIntField:I
```

The above code is in the Smali language, which isn't technically bytecode, but it's a much higher fidelity representation of it than, say, a decompilation or Java. The above code looks like a reference to a field of `ChildClass`, but what if `myIntField` is not defined in `ChildClass` but rather in a parent class? Can you still reference it like this or does it have to be explicit like:

```smali
Lorg/cf/ParentClass;->myIntField:I
```

My intuition is that it makes perfect sense to reference parent fields by the child class which inherits the fields or by the parent class directly. But there could be tons of technical reasons why only reference to the parent class is permitted. Maybe it's ever so slightly faster?

## Testing
The way I tested everything was to create three classes: `ChildClass`, `ParentClass`, and `GrandparentClass`. Create a directory and give it a clever name like `inheri-tests` and place all of these files there.

### ChildClass.smali

```smali
.class public Lchild_class;
.super Lparent_class;

.method public constructor <init>()V
    .locals 0
    .prologue
    .line 3
    invoke-direct {p0}, Lparent_class;-><init>()V
    return-void
.end method

.method public static getsGrandparentFieldLiteral()I
    .locals 1

    sget v0, Lgrandparent_class;->grandparentFieldLiteral:I
    const/4 v0, 0x0
    return v0
.end method

.method public static getsGrandparentField()I
    .locals 1

    sget v0, Lparent_class;->grandparentField:I
    const/4 v0, 0x0
    return v0
.end method
```

### ParentClass.smali

```smali
.class Lparent_class;
.super Lgrandparent_class;

.field public static parentField:I
.field public static fieldLiteral:I = 0x2

.method public constructor <init>()V
    .locals 0
    .prologue
    .line 3
    invoke-direct {p0}, Ljava/lang/grandparent_class;-><init>()V
    return-void
.end method

.method public static getsGrandparentFieldLiteral()I
    .locals 1
    sget v0, Lparent_class;->grandparentFieldLiteral:I
    return v0
.end method
```

### GrandparentClass.smali

```smali
.class Lgrandparent_class;
.super Ljava/lang/Object;

.field public static grandparentField:I
.field public static grandparentFieldLiteral:I = 0x5
.field public static fieldLiteral:I = 0x3

.method public constructor <clinit>()V
    .locals 1

    const/4 v0, 0x4
    sput v0, Lgrandparent_class;->grandparentField:I

    return-void
.end method

.method public constructor <init>()V
    .locals 0
    .prologue
    .line 3
    invoke-direct {p0}, Ljava/lang/Object;-><init>()V
    return-void
.end method
```

### Driver.java

With all three simple Smali classes setup, I needed a Driver class with a `main(String[])` method which could be executed from the command line. For that, I created this:

```java
public class Driver {
  private static class child_class {
    // Dummy class so it'll compile
    static int fieldLiteral = 0;
    static int getsGrandparentField() {
      return 0;
    }
    static int getsGrandparentFieldLiteral() {
      return 0;
    }
  }
  public static void main(String[] argv) {
    child_class.getsGrandparentFieldLiteral();
    System.out.println("field literal: " + child_class.fieldLiteral);
    System.out.println("grandparent field: " + child_class.getsGrandparentField());
    System.out.println("grandparent field literal: " + child_class.getsGrandparentFieldLiteral());
  }
}
```

<div align="center">
![READ IT](/images/understanding-dalvik-static-fields/read-the-code.gif)
**READ THE CODE**
</div>

### Running the Test

Compile `Driver`, remove the dummy child class, and change references to the dummy class to point at our `ChildClass`:

```bash
java2smali Driver.java
rm Driver\$child_class.class
sed -i -e 's/Driver\$//g' Driver.smali
```

For the source for `runsmali` and `java2smali`, check out [Understanding Dalvik Static Fields part 1 of 2](2016/07/31/understanding_dalvik_static_fields_1_of_2/).

Next, compile all of the Smali files into a classes.dex and run the Driver class:

```bash
$ runsmali inheri-tests Driver
  adding: classes.dex (deflated 53%)
[100%] /data/local/runsmali.zip
EXECUTING: Driver
field literal: 2
grandparent field: 4
grandparent field literal: 5
```

### The Results

* `ChildClass` inherited `fieldLiteral` from `ParentClass` even though it was also in `GrandparentClass`. Fields are inherited from the nearest ancestor.
* You can reference fields however you want as long as the object is an instance of whatever is being referenced and has the field defined at runtime. This means maximum ambiguity when parsing, which sucks.

## Fixing Simplify

The way smalivm (the virtual machine powering Simplify) dealt with field inheritance was stupid. It would simply add all fields from all ancestors regardless of public / private to any child class. This was really wasteful of space, but the real kicker is that smalivm wasn't statically initializing (`<clinit>`) super classes when inherited fields were accessed since it wasn't obvious in any data structure where the fields for a class came from.

The naive fix was to keep fields in their defined classes and just be sure to `<clinit>` ancestor classes appropriately. In this case, that means whenever `ChildClass` is instantiated, all of its ancestor classes are statically initialized. But this fix created a lot of complexity. Since child classes no longer had any reference at all to inherited fields, any time a field was accessed, it was often necessary to check ancestor classes, which was expensive.

I sipped my tea and thought about how much I just wanted this damned feature to work and be done with it. I'd been obsessing over fixing this bug since it was originally reported. The solution wasn't in my head. Keep sipping my tea. Still no solution. Maybe it was the two slices of cold pizza I ate while stuck in traffic this morning? Don't judge me. Traffic was horrible. That pizza really took the edge off. Try it. But I think it made my brain slow.

![t and zah](/images/understanding-dalvik-static-fields/tea-and-pizza.png)

Then I got an idea from [Soot](https://sable.github.io/soot/) and [SymDroid](http://www.cs.umd.edu/~jfoster/papers/cs-tr-5022.pdf). Soot can convert Java bytecode to a simpler form they call [jimple](http://www.sable.mcgill.ca/publications/techreports/sable-tr-1998-4.ps) and SymDroid uses something similar called _µ-Dalvik_. The idea is that you can simplify bytecode up front so that later analysis is less complex and easier. Smalivm does something like this already in that every binary math operation such as add, subtract, multiply, etc. are handled by one op internally: [BinaryMathOp](https://github.com/CalebFenton/simplify/blob/master/smalivm/src/main/java/org/cf/smalivm/opcode/BinaryMathOp.java). In this case, for field references, I could simply resolve all references such that they refer to the defining class when building the initial graph of the method. This way, the execution can assume any field reference is pointing at the defining class, and extra lookups to ancestors won't be necessary.

I hope you've enjoyed this two part series which I wrote by painstakingly reassembling scattered notes and code fragments taken while suffering what appears to have been a severe fever dream hallucination. Hope it was educational!
