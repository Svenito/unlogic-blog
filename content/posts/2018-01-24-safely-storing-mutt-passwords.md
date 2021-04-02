---
title: Safely storing passwords with mutt
date: 2018-01-24T14:16:50Z
tags:
  - linux
summary: This post is a quick reminder post for myself, on how to safely store passwords with mutt and keybase (or gpg)
---

If you have either very complex passwords, or you can't be bothered to type them into mutt each session, then you
will be happy to know you can store them safely. Here's what needs to be done.

First ensure you have some keys for Keybase or GPG, then create the following file:

```
$ vim .mutt/passwords

set imap_pass="password"
set smtp_pass="password"
```

Now encrypt it with keybase or GPG and then delete the original, like so

```
$ gpg -r your.email@example.com -e ~/.mutt/passwords
$ keybase encrypt you-username -i ~/.mutt/passwords -o ~/.mutt/passwords.keybase
$ shred -n 5 -z -u ~/.mutt/passwords
```

Finally, add the following line to your `.muttrc`, selecting the right one for gpg or keybase

```
source "gpg -d ~/.mutt/passwords.gpg |"
# OR
source "keybase decrypt -i ~/.mutt/passwords.keybase |"
```

Now you won't need to enter your password anymore, and it's not stored in plain text
