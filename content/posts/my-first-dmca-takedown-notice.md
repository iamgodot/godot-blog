---
title: "My First DMCA Takedown"
date: 2022-11-25T19:04:04+08:00
draft: false
categories:
  - Thoughts
keywords:
  - DMCA
  - DMCA takedown notice
---

It's been a while since last post, yet I haven't been idle. As the title tells, this is a record for that first experience. Since DMCA is a US law, here I chose to write in English.

I'm aware many have read my blog for just one reason: "Amazon OA 2022". I shared quite a bit of information in that post which I consider would be useful for preparing the Amazon online assessment, and there have been some positive feedbacks. Sadly this will no longer be the case.

I published that on 2022.8.11, and removed it permanently on 2022.11.25, which was yesterday.

The reason is an official complaint from Amazon via GitHub where I host this little site. I was a bit surprised though since there are tutorials about Amazon interviews all over the Internet, however, this notice probably implied the one I wrote has been helpful. Anyway, long story short, I complied with the instructions from GitHub and got my blog back online, which all happened in a week.

Well, what I've firmly believed is we can always learn from things. So here I'd like to talk a bit about  this DMCA issue.

> The **Digital Millennium Copyright Act** (**DMCA**) is a 1998 United States [copyright](https://en.wikipedia.org/wiki/Copyright) [law](https://en.wikipedia.org/wiki/Law) that implements two 1996 treaties of the [World Intellectual Property Organization](https://en.wikipedia.org/wiki/World_Intellectual_Property_Organization) (WIPO). It criminalizes production and dissemination of technology, devices, or services intended to circumvent measures that control access to [copyrighted](https://en.wikipedia.org/wiki/Copyright) works (commonly known as [digital rights management](https://en.wikipedia.org/wiki/Digital_rights_management) or DRM). It also criminalizes the act of circumventing an [access control](https://en.wikipedia.org/wiki/Access_control), whether or not there is actual [infringement of copyright](https://en.wikipedia.org/wiki/Copyright_infringement) itself. In addition, the DMCA heightens the penalties for copyright infringement on the [Internet](https://en.wikipedia.org/wiki/Internet).

Wikipedia taught me the above while this page on GitHub seems more relevant and practical: [DMCA Takedown Policy](https://docs.github.com/en/site-policy/content-removal-policies/dmca-takedown-policy). Essentially they think my post was an infringement of their copyrighted content, in terms of some interview questions. So they issued a notice against me, which is transparent to public and can be viewed [here](https://github.com/github/dmca/blob/master/2022/11/2022-11-14-amazon.md). So my next step could be either of:

1. Make changes or remove the content.
2. Argue with a counter notice.

It was an easy decision for me, but instead of notifying me with 1-day time window, GitHub took my site down directly. Then I had to argue to make them give me extra time before the whole thing sorted out.

So if you encounter something like this, there's no need to panic, all you need to do is contact them and make the changes. See [here](https://docs.github.com/en/site-policy/content-removal-policies/guide-to-submitting-a-dmca-counter-notice) for counter notices if you like to fight back. 

Additionally, I wanna mention about removing contents from a Git repository. GitHub has a comprehensive [guide](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/removing-sensitive-data-from-a-repository), where they introduced a tool called `git filter-repo`. My understanding of it's a better substitute for `git filter-branch`, which is not recommended anymore due to some safety issues. So essentially what it does is to rewrite the whole history by eliminating related files as well as the after empty commits, and updating all the commit hashes.

I didn't use it for all I needed was to ditch two simple commits and force push. Nonetheless, this `filter-repo` would really come in handy if you're facing a collaborative repository with a messy history and tons of commits.

---

Anyhow, copyright is vital for all of us, whether on Internet or not. I respect it and I hope this small article could be of assistance to anyone, or simply an eye-opener. :)
