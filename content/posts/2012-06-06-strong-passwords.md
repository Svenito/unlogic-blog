---
title: Strong passwords
date: 2012-06-06T21:03:50Z
---

As I'm just going around and updating the passwords to some of my online accounts, which was prompted by [this](https://translate.google.com/translate?hl=en&sl=no&tl=en&u=https://www.dagensit.no/article2411857.ece) article, I was wonderingwhat a good password really is. Not just in terms of security but also in terms of user friendliness.

I use [Lastpass](https://www.lastpass.com) to manage my passwords and have it auto fill in my credentials on various sites. This works well until I need to manually enter them on another device (iPhone for example - I haven't got a premium subscription yet) or have to type it in just because.

We know that everywhere suggests some wonderfully random characters of at least 8 characters in length and Lastpass actually provides a tool to generate these:

- k%U94\*7r
- 66ds}9R
- 9^wtH7xo

Here we have some 8 character examples for apparently fairly secure options. Fine. They probably are secure. But do you really want to type those in using a touch keyboard on a phone?

Now here's my suggestion: **Three or more random, unrelated words making one password.** For example:

- tarnishedmoleclouds
- refriedchutneygarbage
- turkeyloadedparasol

Those of you who read [XKCD](https://xkcd.com/936/) should already be on the same page as me. Not only are these more memorable (which if you use a password manager is irrelevant) but also much easier to type in on any type of keyboard. But that's not all. Let's have a look at bruteforce times using [these tables](https://www.lockdown.co.uk/?pg=combi)

In the first case we have 8 character passwords made up of "Mixed upper and lower case alphabet and common symbols.". According to the relevant table (and we are assuming a reasonably competent team of crackers using a [class E](https://www.lockdown.co.uk/?pg=combi#classE) attack) these passwords can be cracked in 346days. Not too bad really. But let's see how that compares to the full alphabet, either upper or lower case (not both in this case). Picking one password from above (refriedchutneygarbage) with 21 characters it would take at least 6.3trillions years to break. Much better. Heck, even with 1000,000,000 guesses per second you're still looking at 631billion years.

I can't say how much of an impact adding/removing spaces has on the timings though - if any one knows, or has any insights, do share in the comments. Is using spaces better, the same, or worse than not using spaces? Theoretically I'd imagine that it'd be better with spaces as that's an extra character to add to the list.

## UPDATE

Having said the above, it's worth also assuming that any competent cracker will be using wordlists too, and not just brute forcing the password. According to the [Oxford English Dictionary](https://oxforddictionaries.com/words/how-many-words-are-there-in-the-english-language) there are about 140,000 words in use. This number is a bit high as most of us won't know, or use them all. So to get a more realistic number, I've looked at some common wordlists you can find on the internet and the word count we're looking at is around 50,000 to 70,000.

Given that (as mentioned in the comments) we essentially have a 3 character password with a much larger search space, we can do the math. Three words, 60,000 (taking the middle word count) words will give us 216trillion possibilities. At a rate of 10,000,000 passwords per second it would take around 250 days to crack. Not too shabby still, but not as good as initially hoped, and also worse than your 'messy' passwords.

So my solution to generating secure, but still easily typeable passwords, is to generate 15-20 character long passwords made only of random alphabetical characters rather than words. In this case the figures above will still hold true.

As a quick sampler of a generated password with 16 characters, lowercase only, and pronouncable:

- alicattervetonstou
- molyciontivenzagol
- audentophitendowdy

harder to remember, harder to crack but still easily typed out even with an onscreen keyboard. The pronounceable part is optional but it might aid in remembering the password better.
