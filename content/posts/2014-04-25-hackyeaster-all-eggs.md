---
extends: default.liquid
title: Hackyeaster All Eggs
date: 2014-04-25T21:03:50Z
---

I started writing these up as separate posts, but then I figured it'd be neater to post them all in one.

So here you go, all the Hacky Easter eggs. Some answers will guide you directly to the eggs, others will require a little bit of work to achieve it. But if you got stuck and are wondering where those missing eggs are, I hope this guide will help you.

## Egg 0x01 - Just for fun

The the first egg is fairly easy. There's two ways to approach this one, but ultimately we need to stop that egg from spinning so we can scan it. I first thought it was a .gif file and so opened it in a new tab so I can download it and open it in Gimp or something. Turns out it's a .png, so you can either _Open Image in New Tab_ or save it and view it from there to scan the QR code, or remove the CSS class _eggImage pulse_ from the `img` tag.

Scan the code and we're done.

## Egg 0x02 - Dude, where's challenge 2?!

This egg is a little trickier to get, only because it involves making some mental connections. First off we have no link, simply the title _Challenge 2: Dude, where's challenge 2? It should be after challenge 1._ So where is it? First of all you'll want to do the most obvious thing, that is enter *https://hackyeaster.hacking-lab.com/hackyeaster/challenges/challenge-two.html* in the address bar.

So we've done that, we get a 404. Darn. Let's try *https://hackyeaster.hacking-lab.com/hackyeaster/challenges/challenge_two.html*. Another 404. But something is a little weird. Take a look at the 404 from the previous try, it's in Icelandic (or so Chrome tells us). We also notice that the _-two_ in our incorrect URL is in red - this doesn't happen in the english 404 page. So armed with this knowledge we can perhaps try the Icelandic word for two instead. Google translate tells us this is _tveir_. Substituting this for _two_ in our URL works. We have egg 0x02

## Egg 0x03 - Whooo Whooo!

This egg is found on the app. Once you open the challenge you will see a kid's drawing of an owl and some random letters at the bottom.

Where have we seen letters like that before? Everywhere where you have URL shorteners really. But which one could it be? Well the image gives us a clue: [ow.ly](https://ow.ly)

Collect the egg.

## Egg 0x04 - Nothing to see here

The page and image tells us that there's nothing to see here. Oh well, let's go back and forget about it.

No, let's not, let's just do the usual thing of looking at the page source. It's always a good idea to poke around inside things to figure out how they work. Sometimes they might reveal some useful information.

And there it is... just below the `footer-wrapper` is our easter egg. Albeit in a little less scannable format. Not to worry, let's figure this out. It looks like raw, base64 image data. Two ways to approach this one too. Both achieve the same thing, that is to decode base64 data.

The simplest way to do it is to use an online tool like [the motobit decoder](https://www.motobit.com/util/base64-decoder-encoder.asp) . Simply paste the text into the field, select **decode** then **export to a binary file** and put whatever filename you want with a `.png` extension. Then click **Convert** and your image will download.

Alternatively create a new HTML file and in the header add:

```css
body {
  background-image: url("data:image/png;base64,<base64 data follows here>");
}
```

making sure you add the `body` tags. View the page in your browser and scan one of the eggs.

## Egg 0x05 - Pet Shop

This is less of a brain teaser. You just need to find the QR code in the image. You can do it by just looking at the image and seeing if you can spot it, or start tweaking colour values and then brightness contrast. Eventually you will reveal a QR code. I needed to re-trace some of it in order to get it to scan, but it depends on how well you manage to isolate its colours.

## Egg 0x06 - Australia

This challenge is in the phone application. Click on it and we get a picture of Australia. We all know one thing about Australia. No, not that, the other thing. That's right, they're all upside down. Turn your phone over and screencap the egg to scan it with your app from your desktop.

## Egg 0x07 - E(gg)-Mail

This one had me too. I decoded the obvious base64 message, but then was stumped as to how to extract the eggs. A friend of mine (the one with the windows box) managed to open it in a _.msg_ viewer and get the attachment that way. A _.msg_ is an Outlook/MS mail format. So a viewer to open the file is all that was needed to get the egg.

## Egg 0x08 - _hidden_

This is a hidden egg. But where could it be hiding? Well, have you tried _searching_ for it?

At the time of writing searching for _hackyeaster_ in Google images shows you egg 0x08. It's actually in the screen shots for the apps. So either go to the AppStore or Google Play and look for the hacky easter app. Check the screen shots and you will find egg 0x08.

## Egg 0x09 - Wise Rabbit

The Wise Rabbit.. oh he's such a smug bastard. This is a really annoying one because it's quite easy, but I had to spend quite some time thinking about it. This is where those many hours spent playing text adventures pays off.

So what do we notice about this page? First off it tells us to check the green eggs. First I skimmed the pages again, re-checked all the files and turned up with nothing. "Quick response" - might this relate to the email challenge? Check again but nothing.

*Q*uick *R*esponse. QR. So let's just scan the green eggs again, but this time without the app, but with a separate QR code reader. **Beep** sure enough, there seems to be something there. Scan a few eggs and the pass phrase will reveal itself.

## Egg 0x0a - _hidden_

Another hidden one? I have to admit, these bugged me a lot, because you just had to be nosey enough to find them, and they were quite tricky to find.

So the first clue is that the hidden eggs might be in the files provided, outside of challenges. Where could that be. Well let's have a look at what files they give us. A PDF Flyer. Let's have a look there. Nothing. Just a bunch of blank eggs. But wait, what's that just poking out between those other eggs? Hrmm. Seems to be a little bit of barcode there. I wonder if we can extract that egg somehow. I tried [extractpdf.com](https://www.extractpdf.com/) and sure enough, there's the egg for you to scan.

## Egg 0x0b - I frame, you frame

Another app based one. Fire up the challenge and you have to press a button. You press it and it tells you that it can't be viewed in a mobile browser.

So we need to figure out what URL is being accessed and enter that into a browser. The easiest way is to connect to a proxy and look at the requests in order to determine the URL.

I used OSX and Linux to solve the challenges, and there's a nice Squid frontend for OSX called [Squidman](https://squidman.net/squidman/). Start it up and then connect to it with your device. How you do that depends on whether you have an Android or iPhone, but I am sure you can figure it out. Once connected view the Squid logs and press the button. You should now see the URL `GET` request. Gran this URL and paste it into your browser URL. Snap the pic and

## Egg 0x0c - Call Me!

Call me? What do you mean? Who are you? Where are you? will this run up a massive phone bill?

No it won't. Take a look at the image. There's a protocol specified there: _ps.hackyeaster://_ So enter that in your mobile browser (Chrome didn't work for me, but Firefox did) followed by the number.

Voila, egg 0x0C

## Egg 0x0d - _hidden_

Another hidden one.... _sigh_, at least it's the last one. Well ok, let's have a look at some of the other supporting files. We've got the flyer already. Nothing else to download. Except perhaps.... Yes, the app itself.

If you have an iPhone I can't tell you how to do this, but with an android you need to grab the application APK and unzip it, search for PNG images and one of those is the egg.

Phew - that's all the hidden eggs done with.

## Egg 0x0e - Bunny Research

A PDF with some plain text and some, what appears to be, encrypted text. Well clearly the answer is in the encrypted text. But how do we decode it? Let's just read the text around it. Mostly just big words and hot air. Well, given the format of the encypted text it looks like a substitution cipher of some sort. Perhaps Caesar cipher? let's Google some substituion cipher types.... [Wikipedia has a page on these ciphers](https://en.wikipedia.org/wiki/Substitution_cipher) and after reading a while we notice a word we only just saw **Vigenère**. Well that is a big clue. Let's take a look at [how it works](https://en.wikipedia.org/wiki/Vigen%C3%A8re_cipher). So basically we need to work out the key from the text we have. How can we possibly do that?

Let's not be lazy and write or download some code that will translate between cipher text and plain text for us. I used [this Python script](https://gurno.com/adam/vigen/) to process the text. So let's start analysing the text for something we might be able to identify. For me it was the text **XHV R\* TYNEAP**. Given it's a computer text I bet the plain text is **the A\* search**.

In order to decipher the text and get the possible keyword we need to setup something like this:

```sourceCode
-----------------------------------------------------
| T | H | E |   | A | * |   | S | E | A | R | C | H |
-----------------------------------------------------
| X | H | V |   | R | * |   | T | Y | N | E | A | P |
=====================================================
|   |   |   |   |   |   |   |   |   |   |   |   |   |
-----------------------------------------------------
```

The top row is the plain text we expect, the second row is the cipher text and the bottom row the key (or part of it). In order to get the key we reverse the lookup into the vignere table. Look for the expected plaintext letter in the top row and go down until you find the matching cipher letter. Then look across to find the key's letter.

```sourceCode
-----------------------------------------------------
| T | H | E |   | A | * |   | S | E | A | R | C | H |
-----------------------------------------------------
| X | H | V |   | R | * |   | T | Y | N | E | A | P |
=====================================================
| E | A | R |   | R | * |   | B | U | N | N | Y | I |
-----------------------------------------------------
```

We're getting something here, specifically **BUNNYI**. The **THE** looks like a wrong guess. Let's try another possible 3 letter word:

```sourceCode
-------------
| F | O | R |
-------------
| X | H | V |
=============
| S | T | E |
-------------
```

This gives us **EASTERBUNNY** - **much** more likely to be correct. So let's copy out the text from the pdf into a text file, modify the vigen script to open that instead and just enter what we have so far. One thing you have to do is either modify the vigen script to skip non alpha chars, or remove them from the source text.

Our first try outputs: _THEINVESTIGAHDALITLV...._ So we know that we were lucky that our known text is the beginning of the cipher key. As you can see the text is clearly _the investigation..._. Knowing this you can use the same approach as above to solve the rest of the key. Once you've deciphered the whole text, you get the pass code. Enter it into the page to get your key.

## Egg 0x0f - Paper Chase

This one already provides a big hint: Google maps.

So let's do a quick search to see if we can find the restaurant on line, get its address and take a look. It won't take long to get the location of the restaurant, so open Google Earth and take a look. Lots of photos. None of them the one we want. Clearly we can just search all the photos, but there's a smarter way. The image was taken somewhere right? So let's take a peek at the EXIF data. There we go, location data. Enter the data into Google Earth and we go directly to a photo set.

And sure enough, there's the photo we are looking for. Click, zoom, ehance, scan.

## Egg 0x10 - Broken Egg

We're given two png files. One is a partial egg, the other one doesn't even show. One thing I tend to do is always look at the files in hex if in doubt. The first egg looks normal. Let's take a look at the second egg. Oh, there's some familiar data there at the end. Looks like that base 64 stuff we've seen before. Covert it and save it as a .png file. We get the right half of the egg. Weird. The partial in the second pic is the top half.

Let's take a closer look at the first egg. Notice anything? The header is corrupt. Fix the problem, save the file, and then use your favourite image editor to combine the two halves.

## Egg 0x11 - Number Cracker

We're told what to do, so let's do that

```sourceCode
$> netcat hackyeaster.hacking-lab.com 1234
Enter your guess, dude:
123456789
I need 20 digits, dude!
```

Ok, so not enough digits, let's try 20

```sourceCode
$> netcat hackyeaster.hacking-lab.com 1234
Enter your guess, dude:
12345678901234567890
0<
```

Hrmm. What does that mean? Let's change the command a little to make reentering the numbers easier

```sourceCode
$> echo 12345678901234567890| netcat hackyeaster.hacking-lab.com 1234
Enter your guess, dude:
0<
```

Let's the edit the input from the first number.

```sourceCode
$> echo 22345678901234567890| netcat hackyeaster.hacking-lab.com 1234
Enter your guess, dude:
1<
```

Ahhh so it seems that the first digit tells us how many numbers are correct from the start of the number. The _&lt;_ tells us we're lower than we need to be.

So you have a choice, script it, or, if you are lazy like me, go through it manually with a binary search for each digit. It took a few minutes for me to do it manually (if that), probably about the same time as it would have taken to write a script to do it. Sometimes it's just easier to not try to automate it. Once done, you'll get your egg.

## Egg 0x12 - Lost in Transformation

We've got a wall of text here and our biggest clue is that the beginning of the text \[100:b64\] _b64_ most likely hints at base64 encoding, so let's give it a whirl. Copy everything but the bits in the \[\] and decode.

Now we have `[99:inv][98:URL]`.... oh god. It looks like... yeah, it's been encoded 100 times with a different method each time. We will have to script this. If you fancy the exercise you can run a loop over the text and decode it with whatever method is specified in the \[\]. Each time you come across a new encoding method, add it to the list.

Alternatively view [my gist](https://gist.github.com/Svenito/d28572d6c9a4c1a1b603#file-egg0x0e-py) of it here.

## Egg 0x13 - Tap The Xap

I hated this one. It was a pain. I thought that you had to run the xap to solve this, but I got lucky. VERY lucky. So apparently xap files are just zip files so `unzip TapTheXap.xap` will extract all there is inside the app. Unfortunately no egg.png - that'd be too easy right? Where could it be? I got desperate and ran `grep egg *` on the files. Turns out TapTheXap.dll matches. Ok, let's see what's inside by viewing the hexdump.

Right, so in there is an egg13.png, inside, what appears to be a PKZIP. So I guess the dll contains a zip file with the egg in it. I'll give you a quick tip: you can unzip a zip file inside a dll by just doing `unzip TapTheXap.dll`. I didn't know that at first, but anyway. Now it wants a password. Grrr. The password must be in the app somewhere, so I just ran `strings TapTheXap.dll` to see what we've got. This looks promising _part 1: Dpbwob2HGo_. So there must be a part 2... ah there is. But it has no password. I wish I could tell you I had a really cunning plan and found the second part that way, but I view the file in hex again and searched for part 2 in it. Then I scrolled looking for any interesting data. Eventually I found `tapthexap.zip` followed by a potential password. What else can I do but just give it a try. So add that part to the end of part1 and.... Success!!

## Egg 0x14 - Boolean 101

Here we are given 4 files, all with binary numbers inside. The image actually gives us instructions. What we need to do here is take the data and perform the operations on it. I used a quick Python script to parse the files, remove the white space and perform the required operations. Once you've got that data save it to a new file and learn about [Netbpm](https://netpbm.sourceforge.net/). Use this to generate the QR code.

## Egg 0x15 - Jurrasic Hack

Big clue - Steganography. But which image. I figured out the image by just looking at the file sizes. Alternatively the headers of the images (when viewed in hex) have big clue too. So I found out that the stegasaurus was suspiciously larger than the others. That must be our candidate.

I spent ages trying different steno apps on OSX and linux, but nothing. I must be wrong I thought. So I opened it up in a hex editor to see if that will give me some answers. There was something about _Puff_ - I did a search for it and there's a Windows steno tool call OpenPuff. I got a friend to download and run it and sure enough, the egg's inside.

## Egg 0x16 - Time To Travel

We need to go to where? Well relax. We don't really. Now, if you are on iPhone, I can't help you specifically, but on Android you have to choices again. You can try a fakeGPS app to spoof the GPS coordinates, or you can just edit the app and upload it to the device again. All you need to do is find where the location coordinates are checked and override them. Get the egg and we're done.

## Egg 0x17 - Egg Safe

This Java application requires us to enter a number to unlock the safe. Ok, can we brute force it? Sure, but it will take a while, so let's approach it a little more cleverly. So we can unzip a .jar file and see what's inside. There we have an EggSafe.class - we can decompile this and take a look at what it does. I used an online tool, but you can use whatever you wish. So we see that it's basically applying different algorithms to each part of the number in order to compare them to known hashes. So what we need to do is _simply_ do the same, but iterate over the numbers 0 to 9999 inclusive. Go ahead, give it a go. If you get stuck you can view [my solution](https://gist.github.com/Svenito/dbad0e83f0d637a5701c#file-getit-java). Run it and enter the numbers into the app to decode the image. I decoded the image to disk because the supplied app wasn't showing the image on my machine for some reason.

## Egg 0x18 - Paper And Pen

I loved this one after a period of frustration. I'll try to outline how I managed to decode this from that to finish. The hint here is that it's a paper and pen cipher, so first I looked up information on those and how they work in principle. None of the ciphers really looked anything like the one we have. I kept at it and looked at what the ciphers do and what their output means. Some use a keyword to encode the plain text, but it'd be impossible to decode without the keyword, and there wasn't anywhere that I could see a keyword. Others had the information encoded inside the cipher text. So let's take a look at the text a little bit closer.

```sourceCode
Dii2 Dii3 Di2 Gi1 Gi1 Aiii1 Diii2 Gi3 Aiii2 Gi2 Giii1 Dii3 Aiii3 Gii3 Di2 Diii3`
```

We can see a few things here. The initial letter is either _A_, _D_, or _G_, then follows one of _i_, _ii_, _iii_ and ends with _1_, _2_, _3_. So that's actually 3 bits of information. Which could mean we're referencing 3D coordinates. What's the deal with A, D, and G though? Oh, look, the are all the same distance apart. So that got me thinking. _ABC_, _DEF_, _GHI_. Then I realised that those bits of information have 27 permutations, enough to chunk the alphabet into 3 groups.

So that's the first group, then the next bits of the alphabet go into a second 3x3 group and finally the last letters. Each components is an index into one part. The letters tell you the line, the _i_'s tell you the chunk and the number the column.

I ended up with this (forgive the crap ASCII table. I'll see about getting a nicer one):

```sourceCode
-----------------------------------------
||    i      ||     ii    ||     iii   ||
-----------------------------------------
|| 1 | 2 | 3 || 1 | 2 | 3 || 1 | 2 | 3 ||
=========================================
|| A | B | C || J | K | L || S | T | U ||
-----------------------------------------
|| D | E | F || M | N | O || V | W | X ||
-----------------------------------------
|| G | H | I || P | Q | R || Y | Z |   ||
-----------------------------------------
```

So _Dii2_ translates to `n`. Use the table to get the full text
