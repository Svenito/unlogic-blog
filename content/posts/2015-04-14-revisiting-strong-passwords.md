---
extends: default.liquid
title: Revisiting strong passwords
date: 2015-04-14T21:03:50Z
---

Some time ago I wrote a post about [strong passwords](https://unlogic.co.uk/2012/06/06/strong-passwords/). That was three years ago and I figured I might as well revist what I wrote back then.

Since then technology has moved on a lot, and the number of password breaches has increased, aswell as the number of reports mocking our poorly chosen passwords.

As far as I am concerned, I am still using 15-18 random character passwords. I'm no longer lowercase only, but a mixture of upper, lower, digit, and punctuation. The length depends on how long I am allowed to make my passwords. Believe it or not, some sites limit you to a maximum length of 12, possibly less on some sites. Silly huh? Not only that, but still they tell me my 18 character password without punctuation or digits is less secure than a 4 character mixed character password. Hmph.

But what is cropping up more and more, and what I wanted to write about, is the password rules on signup pages. For example "Your password must be at least 8 characters long, contain one upper case letter, and a number." Sometimes a _special character_ is thrown into the mix too. The issue here is, that although your final password is more secure (in theory), the search space for a valid password is reduced. With some attackers being able to generate 1 trillion guesses per second, keeping the size of the search space large will help.

By how much difference does it make? Well that's what I want to figure out.

So using the commonly used english alphabet with digits and special characters we have the following available

```sourceCode
Name    | Content                           | Character Count
--------|-----------------------------------|-----------------
lower   | abcdefghijklmnopqrstuvwxyz        | 26
upper   | ABCDEFGHIJKLMNOPQRSTUVWXYZ        | 26
digits  | 0123456789                        | 10
special | !"#$%&'()*+,-./:;<=>?@[\]^_{\|}~` | 32
```

Let's assume that I have a list of password hashes from somewhere. I know that the password is exactly 6 characters long (which I think is a reasonably common password length these days), and I also know what rules govern the choice of password when it is created. We'll look at these rules in turn and see how much difference they make. I fix the password length so I don't introduce too many variables.

I'll be basing the calculations of a few assumptions:

- Hashes are SHA256
- We have a _reasonable_ PC available (1x NVidia gtx580), managing 355 Mh/s (355,000,000 hashes/s) ([ref](https://hashcat.net/oclhashcat/#performance))
- These are pure bruteforce attacks. No wordlists, permutation or combination attacks
- Timings for each attack assume _worst case_. i.e. we have to run through all guesses. Usually an attack stops when a valid match is found, shortening the attack.

Let's analyse how different password creation rules affect the duration of the attack.

For reference, these are the number of possible combinations for each set of characters

```sourceCode
Possible characters           | Character count | Number of combinations
------------------------------|-----------------|-----------------------
lower only                    | 26              | 308,915,776
upper and lower               | 52              | 19,770,609,664
upper, lower, digits          | 62              | 56,800,235,584
upper, lower, digits, special | 94              | 689,869,781,056
```

## No rules

Using any combination of characters the number of possible passwords is _689,869,781,056_. It would take _1943.29 seconds_ (689,869,781,056 / 355,000,000) to crack this password. That's just over half an hour.

_Attack time_: 1943.29seconds

## At least one upper case

If we are forced to chose at least one upper case character, we are also saying that there are no passwords now with just lowercase characters. The number of possible combinations is now _689,869,781,056 - 308,915,776 = 689,560,865,280_, or _99%_ of the original search space. This is a small impact of only 1 second.

_Attack time_: 1942.42seconds

## At least one upper case and one digit

Now we know that there are no passwords with just lowercase, or with lower and uppercase only. Therefore we can also remove these from the list of possibilities. Our new number is now _689,869,781,056 - 308,915,776 - 19,770,609,664 = 669,790,255,616_ or _97%_ of our search space. Now we're starting to see savings of up around 100seconds. In the grand scheme of things, still not much

_Attack time_: 1886.73seconds

## Must contain all of the above

Upper, lower, digits, and special all need to be present. Therefore we can remove all the other possiblities for a grand total of: _689,869,781,056 - 308,915,776 - 19,770,609,664 - 56,800,235,584 = 612,990,020,032_ or _88%_ of the original. Now we've saved another 100 seconds.

_Attack time_: 1726.73seconds

## Conclusion

Although contrived, this scenario should indicate that _forcing_ people to do things in the interest of security can help attackers too. If we are allowed to use any character out of the full set, an attack would have taken 4 minutes more than if we are forced to create a _secure_ password that _must_ use certain characters. The difference isn't much if you look above, but bear in mind that the longest attack is _1.12_ times longer. If we extrapolate this to a 8 character password, it's a difference of almost 2 years (51.18 vs 49.7).

Needless to say, knowing the minimum length reduces the search space once again, because now I won't even bother with anything below 8 characters. That's a fairly big chunk of possibilities.

The times can be further optimised by employing wordlists, known substitutions and other rules. The more you know about the nature of the password, the less time it takes to crack it. Yes, applying rules and substitutions to wordlists takes time, but it's insignificant to the amount of time they can shave off of a brute force attack. And the more an attacker knows about the nature and composition of your password, the better they can tailor their wordlists.

Even [Edward Snowden's advice isn't bulletrpoof](https://www.wired.com/2015/04/snowden-sexy-margaret-thatcher-password-isnt-so-sexy/)

The key really is not to force people to have a specific password combination, but to encourage good password creation. Long, random, and unpredictable, passwords from a large vat of possibilities.

Don't tell attackers what the password isn't. Let them guess.
