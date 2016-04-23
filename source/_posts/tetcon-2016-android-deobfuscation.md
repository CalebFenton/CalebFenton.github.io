title: "TetCon 2016 - Android Deobfuscation: Tools and Techniques"
tags:
  - dex-oracle
  - simplify
  - deobfuscation
  - android
comments: true
date: 2016-04-23 00:00
---

I gave a talk at [TetCon 2016](https://tetcon.org) about Android obfuscation and deobfuscation.

The talks at TetCon were great and the people there were super nice. I got all kinds of new ideas and spent the entire flight home furiously coding. Super motivating to hear from and talk to other people working on similar problems. Thanks to the organizers and volunteers Thai for making everything happen.

Also, special thanks to everyone for speaking English around me!

# Slides + Video

http://www.slideshare.net/tekproxy/tetcon-2016

# Tools

* https://github.com/CalebFenton/dex-oracle
* https://github.com/CalebFenton/simplify

# Abstract

State of android deobfuscation is weak. Commercial obfuscators are getting more common, and reversers need to understand how to deobfuscate them. This talk provides an overview of different obfuscation types. After that, it describes two code deobfuscation approaches: pattern recognition and virtual execution.

Pattern recognition focuses mainly on identifying obfuscation patterns, crafting into regular expressions, and then repeatedly applying pattern-based transformations on the code. Insight into code behavior is improved by limited execution of certain methods and storing the result.

Virtual execution involves simulating the applications code to determine semantics. A context sensitive graph is generated representing every possible execution path and all possible register + class states for each execution of each instruction. This is then analyzed and modified to make the code easier to understand but behaves identically.
