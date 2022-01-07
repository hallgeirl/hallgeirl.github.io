---
title:  "Time to block those third-party tracking cookies"
date:   2020-08-11 15:00
categories: privacy
redirect_from:
  - /2020/08/11/time-to-block.html
---

Ironically, my first "real" post on this blog won't be related to coding. But I figured it's quite relevant these days, and I wanted to share it, if for nothing else as a PSA.

More or less every social media website, and TONS of other websites and advertisement companies, use tracking cookies to monitor user behavior. Usually this is considered quite harmless, and the cookies only identify that when you visit site A, then site B, that it's the same user visiting both. Neither site A or site B knows that your name is Bob. Then the owner of the cookie (e.g. Google) can tie that tracking cookie to your account to serve you personalized ads.

However, sometimes you may be asked for an e-mail address, your name and other information on a website. Perhaps you're creating an account there, perhaps you're signing up for an e-mail list. Suddenly the website you're visiting have both the anonymous tracking cookie, AND information to connect to it. This information can be sold to third-parties, and it is.

Yesterday I learned that there exists services out there, like one called GetEmails, that lets websites identify anonymous web traffic and connect your **anonymous** visit to your e-mail address, your name, home address, phone number and more, for the site owners to use, based on tracking cookies. 

Let that sink in... If you visit a site as an anonymous user, and that website uses this service, that website may get access to your personal contact information including your **home address**! That means they can both call you on your phone, send you physical mail, and in principle also sell that information to others as well. And I see no reason why scammers also can make use of this. Visit evil.com, and suddenly the scammer has your home address. If it doesn't freak you out, it should. This is real.

Now, thankfully there's ways to block tracking cookies. Some browsers do it by default - for instance [Firefox blocks a lot of tracking cookies out of the box](https://support.mozilla.org/en-US/kb/disable-third-party-cookies). Today I switched Firefox from the standard blocking mode, to strict. For those not using Firefox, most modern browsers do support blocking tracking cookies. 

If you're interested in hearing a real-life story about this, I can recommend an episode from one of the podcasts I listen to, [Smashing Security, episode 190](https://www.smashingsecurity.com/190). It's an entertaining and very relevant podcast these days.

**Tl;dr: You should block third-party cookies, also known as tracking cookies!**