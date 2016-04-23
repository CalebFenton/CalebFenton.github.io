title: Why Most Vulnerabilities Are Never Disclosed
tags:
  - security
  - open source
comments: true
date: 2016-04-29 00:00
---

When it comes to writing software, humans are the best game in town. Unfortunately, we're absolutely _terrible_ at it. Of course, we're good at other stuff -- recognizing faces, tool use, gossiping, and bi-pedal locomotion, but it turns out our brains are not so good at giving a computer the thousands of tiny, precise instructions necessary to [validate an email address](http://www.ex-parrot.com/~pdw/Mail-RFC822-Address.html) or [properly deal with names](http://www.kalzumeus.com/2010/06/17/falsehoods-programmers-believe-about-names/). That fact we get anything to work at all is amazing

The bottom line is that if developers are writing code, they're writing bugs and some bugs are vulnerabilities. Some are found and responsibly disclosed while others are kept secret or [sold](http://0day.today/). For reasons which I shall explain, I believe that _most_ security vulnerabilities are fixed but _never disclosed_.

<!-- more -->

## Why does disclosure matter?

It all has to do with getting people and businesses to update. The cost of updating a library can be [quite high](/2016/04/22/reversing-an-open-source-vulnerability), especially at larger organizations. It's not a good value to update if only to get a few more features and a few less bugs but getting _owned_ [is](https://www.privacyandsecuritymatters.com/2015/02/target-data-breach-price-tag-252-million-and-counting/) [expensive](http://www.networkworld.com/article/2879814/data-center/sony-hack-cost-15-million-but-earnings-unaffected.html) and it makes perfect sense to update if it means not getting owned.

If most vulnerabilities are never disclosed, businesses may neglect to update vulnerable code. Attackers could exploit this by searching for fixed but undisclosed vulnerabilities.

## Why aren't vulnerabilities disclosed?

There are several forces at play which might cause a vulnerability to slip through the cracks.

### Overlooked Bug

The most obvious possibility is that the security implications of a bug weren't obvious when the developer fixed it. I have no idea of knowing how common this is relative to the other reasons but it has to happen. Developers can't be expected to do a full security audit for every bug they fix. Even if there was some review system in place, figuring out how to exploit a bug is _highly non-trivial_, especially for a large or complex app.

### Lack of Awareness

A developer may not be aware of how important it is to disclose every vulnerability. It's not immediately obvious why it's important, especially if you're an inexperienced developer. I believe this is the most common reason for undisclosed vulnerability fixes. In my experience, this is most common in trendy, new projects, i.e.  NPM modules and Ruby gems. Mature projects run by established, organized development communities are more likely to already be familiar with the importance of disclosure. For example, Apache has a pretty good track record of disclosing any vulnerabilities with public advisories any time there's a release which includes a security fix.

### Lack of Resources

Even if a developer understands the importance of disclosure, they may not have time or desire to jump through all the hoops to do a proper disclosure. Instead, maybe they just mention that they fixed some security issues in their change logs.

A proper disclosure would include requesting a Common Vulnerability and Exposure ID (CVE-ID) or by announcing the issue to a language or framework specific advisory such as [Ruby Advisories](https://github.com/rubysec/ruby-advisory-db) or [Node Security Advisories](https://nodesecurity.io/advisories). Announcing to any of these channels would help to quickly spread the word about a vulnerability, but it requires doing a lot more than writing and committing code and if a developer is maintaining a project in their spare time, they might not be keen to do all this extra work.

There has been some frustration recently with the CVE system due to researchers being denied CVE-IDs for certain types of issues or because CVE-ID requests were never responded to. More info: [Concerns about CVE coverage shrinking](https://cve.mitre.org/data/board/archives/2016-03/msg00002.html) and [Ruminations on App CVEs](https://www.nowsecure.com/blog/2015/09/16/ruminations-on-app-cves/).

### Lack of Humility

Imagine that you spend a lot of time on a side project and enjoy the admiration and respect of your peers. Then one day you find you've made a mistake and didn't notice a vulnerability in your code. Maybe it was a big, dumb mistake, too, and now everyone might be getting owned because of your stupid mistake. It takes a special kind of someone to get over that and admit it openly; it takes humility and bravery. If it's a group of developers, the community needs to be healthy and understanding.

## Summary

I believe this is an important issue for developers and businesses to understand, but I don't think the sky is falling. Companies are increasingly using open source code and this is a security issue people should be thinking about.

We're working on tools that help us collect and identify commits to open source projects which fix undisclosed vulnerabilities. We'll be able to use this information to better protect our users. Initial tests turned up a few dozen vulnerabilities before I had to work on something else. The idea seems sound. I can't mention any specific details right now, but we'll be publishing the results and specific details soon.
