---
title: Passwords
description: This is how I deal with passwords and managing them
date: 2022-09-05
tags:
- password
- password manager
- linux
- passwordstore
- tomb
- encryption
- pgp
- git
---

For the last 7 or 8 years I've been using the Linux terminal password manager
[Pass](https://www.passwordstore.org/). I find this to be very easy to use and it
fits in nicely with my workflow. I have combined this with some other software
that helps in using this across devices.

What you'll need:

- GPG and a PGP Key, [I won't cover setting these up](https://duckduckgo.com/?q=create+a+PGP+key)
- [Pass](https://www.passwordstore.org/)
- [git](https://git-scm.com/)
- [tomb](https://www.dyne.org/software/tomb/)

Make sure to check out the [Pass website](https://www.passwordstore.org/), it
lists many plugins and clients that you can use.

# Pass

Setting up Pass is fairly simple. See the installation docs for your OS to get it
installed and then you would run `pass init [gpg-key-id]`, where `[gpg-key-id]` is
an ID for a PGP key you have full access to, this sets up a new password store
located in `${HOME}/.password-store`. If you then run `pass` you will get some
output similar to the following:

```bash
❯ pass init "Luke Mallon <luke@mallon.ie>"
Password store initialized for Luke Mallon <luke@mallon.ie>
❯ pass
Password Store
```

Congratulation you now have a password manager setup. Let's add our first password,
you can do that with the following command `pass generate [name of file] [length of password]`,
that will look something like this:

```bash
❯ pass generate luke 100
The generated password for luke is:
512XKaSods/0Y8(za"=k`WH#ABfHrVG.(^;MW/Wvb=HVay,swJ[tW'IfDexL"yA]&~-Mp6T1Ek_..{B0DPX|VkLZH$P0HrMcVdIy
```

Then if we run `pass` again we'll see this file listed:

```bash
❯ pass
Password Store
└── luke
```

Now you can work with passwords on one machine, to view your password type `pass [path-to-password]` e.g.:

```bash
❯ pass luke
512XKaSods/0Y8(za"=k`WH#ABfHrVG.(^;MW/Wvb=HVay,swJ[tW'IfDexL"yA]&~-Mp6T1Ek_..{B0DPX|VkLZH$P0HrMcVdIy
```

# Sharing

Now in order to be able to use these passwords across devices you'll want to sync
it in some way. Pass comes with built in support for `git` so you can use that
with a host like [github.com](https://github.com), [gitlab.com](https://gitlab.com)
or any other git host out there, you could even host your own with a light weight
git server like [gitea](https://gitea.com) or [charm_ SoftServe](https://github.com/charmbracelet/soft-serve).
I've set up mine on [keybase](https://keybase.io) (see the [docs for how to](https://book.keybase.io/git#making-a-repository)).

Use the following command to enable git for your password store:

```
❯ pass git init
Initialized empty Git repository in /home/nalum/.password-store/.git/
[main (root-commit) d1200ce] Add current contents of password store.
 2 files changed, 1 insertion(+)
 create mode 100644 .gpg-id
 create mode 100644 luke.gpg
[main 199756e] Configure git repository for gpg file diff.
 1 file changed, 1 insertion(+)
 create mode 100644 .gitattributes
```

You can then setup your remote with `pass git remote add origin [git url to host repo]` e.g.

```bash
❯ pass git remote -v
❯ pass git remote add origin keybase://private/nalum/testing
❯ pass git remote -v
origin  keybase://private/nalum/testing (fetch)
origin  keybase://private/nalum/testing (push)
```

With that setup you can now push your passwords and pull them down on another device.

# Tomb

The last thing we'll look at is setting up Tomb with [`pass-tomb`](https://github.com/roddhjav/pass-tomb#readme).
This is an extention to the `pass` cli that adds some extra functionality to it.
There are a number of ways to [install](https://github.com/roddhjav/pass-tomb#installation)
which I'll leave for you. Once installed you can see the details by running `man pass-tomb`.

So in order to entomb our password store we're going to need to move the password
store from it's current location e.g. `mv ~/.password-store ~/.password-store.bak`,
now we need to create the tomb, which we can do as follows:

```bash
❯ pass tomb "Luke Mallon <luke@mallon.ie>"
 (*) Your password tomb has been created and opened in /home/nalum/.password.
 (*) Password store initialized for Luke, Mallon, <luke@mallon.ie>
  .  Your tomb is: /home/nalum/.password.tomb
  .  Your tomb key is: /home/nalum/.password.key.tomb
  .  You can now use pass as usual.
  .  When finished, close the password tomb using 'pass close'.
```

Now if we list the contents of the directory we'll see the following:

```bash
❯ ls -lah ~/.password-store
total 30K
drwxr-xr-x   3 nalum nalum 1.0K Sep  5 14:35 .
drwx------ 129 nalum nalum  12K Sep  5 14:47 ..
-rw-------   1 nalum nalum   29 Sep  5 14:35 .gpg-id
-rw-------   1 nalum nalum    6 Sep  5 14:35 .host
-rw-------   1 nalum nalum   11 Sep  5 14:35 .last
drwx------   2 nalum nalum  12K Sep  5 14:35 lost+found
-rw-------   1 nalum nalum   11 Sep  5 14:35 .tty
-rw-------   1 nalum nalum    5 Sep  5 14:35 .uid
```

We'll want to copy the contents of our moved Pass directory into this newly created
one e.g. `mv ~/.password-store.bak/* ~/.password-store` and then we can work with
Pass as normal, running `pass open` and `pass close` to open and close the tomb.

```bash
❯ pass close
 (*) Your password tomb has been closed.
  .  Your passwords remain present in /home/nalum/.password.tomb.
❯ pass open
 (*) Your password tomb has been opened in /home/nalum/.password-store/.
  .  You can now use pass as usual.
  .  When finished, close the password tomb using 'pass close'.
```

You can also setup a timer (if you are using a linux distribution with systemd)
so that your password tomb closes automatically e.g. `pass open -t 10m` this will
close the tomb after 10 minutes, it also updates a file `~/.password-store/.timer`
with this timing information:

```bash
❯ pass open -t 10m
 (*) Your password tomb has been opened in /home/nalum/.password-store/.
  .  You can now use pass as usual.
  .  This password store will be closed in 10m
❯ cat ~/.password-store/.timer
10m
```

# Conclusion

Now we have a password manager that is owned and maintained by only you. Set up
in a git repostory that you can push to a hosting provider like github/gitlab/keybase
or you can self host with gitea/charm_ softserve. It's encrypted using your PGP Key
which you will either need to install on all devices that you'll want to decrypt
the passwords with or can be setup on a Yubikey (or similar device) allowing you
to have your private keys locked away on a device that is easily moved around and
not requiring you to have duplicates of your private key on multiple devices.

If there is interest I'll go over what my PGP key set up is and also go over the
setup of clients ([Android Password Store](https://github.com/zeapo/Android-Password-Store#readme),
[passff](https://github.com/jvenant/passff#readme)) that I use with Pass.

