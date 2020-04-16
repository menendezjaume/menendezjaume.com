+++
banner = "images/gocheese-after.webp"
date = "2020-03-28T13:47:08+02:00"
tags = ["Security Awareness"]
title = "gocheese.me - a Security Awareness Tool"
short_description = "A web-based tool to increase security awareness amongst your peers."
+++

In times where most of us are working remotely, I thought it was important to remind us all of the importance of security awareness, an aspect of security that thanks to the protection of our homes may sometimes be neglected.

Security awareness, as put by some [sources](https://en.wikipedia.org/wiki/Security_awareness), is _the knowledge and attitude members of an organisation possess regarding the protection of the assets of that organisation._ 

I really like that definition. Knowledge is easily transferable between individuals in an organisation. You just need to run a workshop, send an email, document something, and you got it. However, a change in attitude is much more challenging to achieve.

When it comes to security, I believe attitude is at least as important as knowledge, and changing old habits demands hard work.

I remember that in one of my old companies, where we were consultants and spent a lot of time on client site, we use to play a game (and you probably played it too). Every time someone in our team left their laptop unlocked and unattended, we would do a _rumbero_ on them (yes, from the rumba dance style). This _rumbero_ consisted of sending an email to the rest of the team with a naughty story or proposition. Sometimes, we would invite everyone for a drink after work. Other times, we would gift something. Some others... well, there would be some banter.

I don't remember when we started it. I am sure it was in place when I joined, and there were times that we wouldn't even think of it as security awareness. The truth is, going back to the definition, that is attitude. 

The fear of getting _rumbero_'d on made us all lock workstations every time we went away from them, which in consequence was a fantastic way to foster security.

At Thought Machine, we have something very similar build by my teammate and amazing security engineer [VJ](https://github.com/VJftw). It tracks metrics, and we use them not to blame or finger point anyone, but provide additional security training to those who seem to need it most.

I love that tool, and in the security team we probably are the biggest offenders of all; not because we don't know about security but because we are always on the lookout for a victim - and who's nearest than a fellow security engineer who got up to grab a coffee?

As a result of the benefits that I've seen come from these practices, I have created [gocheese.me](https://gocheese.me). 

[gocheese.me](gocheese.me) is a tool anyone can use to privately have a laugh with someone who's left their workstation unlocked and unattended without shaming them publicly or belittling them in any way.

{{< figure src="/images/gocheese-before.webp" class="image fit" title="Go Cheese Me (gocheese.me) before cheesing" >}}

It works by visiting the website and confirming that you want to _cheese_ someone from the workstation you are visiting from. Once you click on the _cheese me!_ button, the message changes for one saying that you've been cheesed, and a description of how to lock your workstation next time. Also, to make it a little more fun, the background changes with a random meme.

{{< figure src="/images/gocheese-after.webp" class="image fit" title="Go Cheese Me (gocheese.me) after cheesing" >}}


And, let's not forget, since it's a security tool, I thought I'd share the source code so everyone can see that I'm not doing anything in the background. Also, you can fork it and make it your own!

The tool has been written in ReactJS and is hosted in S3. I am also using a CloudFront distribution not because I thought it was required - the content is minimal - but because I didn't want to add any cookies but still get some metrics. I don't collect any personal information, but rather country of origin and some other meaningless data.

Needless to say, as a Security Engineer specialised in the cloud, I'm naturally more inclined to backend technologies. And even though JS is a language that can be used for both frontend and backend, I am not very familiar with it. Because of it, if you see something not quite right, feel free to open an issue or a PR.

Next time you're in the office if you see an unlocked and unattended workstation, go cheese them; or put a different way, [gocheese.me](https://gocheese.me)