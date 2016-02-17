title: Why Anti-Virus Software Sucks
tags:
  - realtalk
  - anti-virus
comments: true
date: 2016-02-07 20:13:40
---

Everyone knows anti-virus products suck and you can say anti-virus sucks for many different reasons and at different levels. You could start with obvious, surface level reasons: anti-virus software (AV) sucks because it's slow, klunky, self-advertising garbage that slows your machine down. From there, you could move on to more perceptive complaints such as how it hardly ever detects new malware and almost certainly will not detect fancypants, bespoke, advanced persistent threats (APT). You could still deeper and claim that there's something wrong with an industry that thrives on selling people fear and selling companies mere compliance so their insurance doesn't laugh in their faces when they try to collect after getting their gibson's backdoor hacked.

The obvious question is then _why_ do AV products suck? Malware is a big problem that costs people money and heartache all the time. Why isn't this solved better? Need to understand the problem at the most fundamental level. For me, this means understanding the condition in terms of economics principals--incentives, constraints, market forces at work, and so on. Once you understand something at this level, you can usually extrapolate most of the symptoms yourself and, importantly, you'll have a much better idea of how to actually _fix_ it. This brings me to my main thesis: **AV software sucks because it's impossible for the market to be informed and to meaningfully differentiate between products and objectively determine which one is better.** Because of this, there isn't much incentive for companies to make lean, clean, optimized, AV products with amazing, complex detection capabilities and behavior analysis. They can't compete on quality, because people can't tell the difference between great and crap, so they have to compete on sales and advertising.

![fearmongering](http://usercontent1.hubimg.com/3675524_f520.jpg)

You may have a favorite AV scanner, but can you honestly tell me, objectively, why it's better than all the others? You might have a few good, small reasons like one has a cleaner, faster user interface (UI) or _seems_ to have better detections. But how can you be sure? Do you know what each one is really doing under the hood? How do you know how good it is at detecting new viruses? How do you know how fast its detection signatures are updated? What about code quality and security bugs in the AV software itself? If you're like most people, you simply prefer one AV product because it sucks the _least_.

![gear grinding](https://i.imgflip.com/8bki5.jpg)

It's pretty much impossible for laypeeps to have any clue how good an AV product is. Actually, I can't even tell, so I reckon most experts can't either, at least in general. Because of this general inability to compare, AV testing companies have stepped up to solve this problem. They test and compare all the different AV products and claim to provide an objective, meaningful, comparative analysis. The idea is that consumers can read their reports and really know what's going on and pick a good AV product based on _science_ and not anecdotal hearsay.

![difficulty](http://www.kappit.com/img/pics/201412_1752_gaaid_sm.jpg)

## Why AV Testing Sucks

First, if you've never seen an AV Test report and you're some kind of masochist, here are two from 2015:

1. https://www.av-test.org/fileadmin/tests/mobile/avtest_summary_mobile_2015-11.xlsx
2. http://www.av-comparatives.org/wp-content/uploads/2015/12/avc_prot_2015b_en.pdf

They evaluate products on various features and abilities such as:

1. % of malware detected (detection rates)
2. % of good apps detected as malware (false positives)
3. ability to detect new and unknown threats (0day detections)
4. user-interface and usability
5. battery usage (for mobile devices)
6. other nice features: backup, device locate, remote wipe, remote lock, etc.

Sounds good, doesn't it? Where's the problem here?, you may be wondering. I'll just cut straight to the heart of it. Let's start at #1, detection rates, the most critical metric. _How can an AV test determine this?_ How can they possibly know how much malware an AV product detects and misses? First off, they would need a sample set of *known* malware to run through the AV scanner. Hmm. But that can't be right. How can they make a malware sample set? How can they know what malware is? If they had perfect knowledge of what was malware and what wasn't they would *be* an AV company not an AV _testing_ company!

The truth is that AV testing companies have no clue how to detect malware. Instead, and this part is just pure gold, they rely on the _AV companies_ to provide the malware samples. Sort of like if college students got to pick which questions were on the test.

Yeah. Just let that sink in for a minute. While you meditate on how fucked up that is, contemplate on the "appropriate" use of "ironic" quotes:

![ironic quotes](https://i.imgur.com/K8AFzev.png)

### Problem 1: Black Magic

Right off the bat, the objectivity of the test is seriously compromised. Different AV companies submit their samples and somehow the AV test has to somehow figure out which ones are actually malware and which ones are false positives from the AV company. How does it achieve such a feat? I'm assuming it has something to do with many Excel spreadsheets, animal sacrifice, and _fucking witchery_.

There was one time I found an entirely new family of sneaky malware that no one else detected. So we submitted some as test samples. Since everyone _except us_ missed it, they took it out of the sample set. Because, you know, it must not be malware, right?

![its magic](https://i.imgur.com/VAsfyWg.jpg)

### Problem 2: Biased Against Small Companies

Not all companies have the time or resources to curate and submit test samples, so that only leaves the big companies. And you can be absolutely 100% sure the big companies have test machines setup to run samples through the competition so they know exactly what samples the other AV products are likely to miss. Because of this, the sample set is biased against companies which don't bother or can't afford to do "offensive sample testing". I've been at conferences and talked with people enough to learn that some companies spend millions of dollars and have entire teams just for winning these tests. That's money and resources _not_ going to improving the product and protecting the customer.

### Problem 3: Unrealistic Samples

The sample sets used in these tests are completely unrealistic. It's usually a bunch of samples from a few families no one ever actually encounters in the wild. Where I worked, ~0.3% of our test misses were actually encountered by users. That means 99.7% of the test samples were never seen _once_ by _any_ of our customers.

Since the number of families used in the sample set is not nearly representative of the number of actual families spreading in the wild, if an AV company misses one family, they could miss a lot of samples. It's like when one paragraph from a single chapter in a book is used to make 99.7% of the test.

I remember when I saw Microsoft Security Essentials massively fail a particular AV test and was amazed by their badass response:
https://blogs.technet.microsoft.com/mmpc/2014/01/17/key-lessons-learned-from-the-latest-test-results/

In this polished, polite, and diplomatic response, homeboy drops the mic with this little gem:
> Our review showed that 0.0033 percent of our Microsoft Security Essentials and Microsoft Forefront Endpoint Protection customers were impacted by malware samples not detected during the test. In addition, 94 percent of the malware samples not detected during the test didn’t impact our customers.

I don't want to be accused of putting words in anyone's mouth, but this basically says "y'alls test is bunk lol whatever".

### Problem 4: Not All Detections Are Equal

If an app with ads is infected with the most malicious, vile, and insidious malware ever created by man is submitted to AV test as a sample, the tests have no way to distinguish between a detection of the ads and a detection of the actual malware. What you end up with is every product scurrying around to aggressively detect adware. Also, this creates an incentive to create broad, noisy signatures which detect just about anything possibly bad. These "weak detections" usually read like as "This application smells funny and might be bad, but we're not saying it's bad. We don't know. Don't break eye contact! Good luck!".

![unequal](https://i.imgur.com/TZSHLqV.jpg)

## Conclusion

Because good testing is so hard, people will continue to pick what AV product they use based on reading tea leaves, chicken bones, PCMagazine articles, word of mouth, etc. Until quality can be better measured, AV companies will continue focus on marketing and winning these bullshit tests, or pretty much anything except for making their stuff better.

Good testing is needed and that requires a good sample set of relevant, recent, modern malware. But those best equipped for creating this ideal sample set are the same being tested, so a perfect solution may not be possible. One way to improve it would be if companies open sourced their detection data and samples from some of the most prevalent families. I'm not talking a download link on their main page, but if a security researcher or academic wanted a copy, they could contact the company and there would be a system in place to verify they're legit and send them the goods. This would open up AV testing to competition and would also make the process more transparent. Another cool side effect is academics can stop using super old malware sample sets for research.
