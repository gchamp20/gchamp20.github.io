---
layout: post
title: Using git send-email with gnome-keyring
excerpt: Setting up git send-email with gnome-keyring without a login manager.
---

## Problem

I want to use [git send-email](https://git-scm.com/docs/git-send-email) without having to type my mail password (or app password), in an headless scenario. Specifically, when I ssh into my computer without going through a login manager.

Fortunately, fetching credentials with git is highly configurable. This posts details how `gnome-keyring` can be manually unlocked, how to put mail credentials in it and how to fetch them via git.

## Unlocking a keyring

On most distibutions using gnome, everything is setup correctly to unlock the keyring when graphical sessions start. However, none of this works when logging in to a computer using ssh only.

This can be worked around by manually spawning a gnome-keyring-daemon, unlocking it and setting up the environment variable `GNOME_KEYRING_CONTROL` to the correct value.

To spawn a gnome-kerying-daemon, create a start-keyring-daemon script with the following content:

```
#!/user/bin/env bash
read -rsp "linux user password: " pass; echo -n "$pass" | gnome-keyring-daemon --unlock
```

`gnome-keyring-daemon` expects your linux user password on its standard input before starting up, which is why the `read` command is piped into it.

After it has spawned, `gnome-keyring-daemon` creates its control socket at `$XDG_RUNTIME_DIR/keyring`. `XDG_RUNTIME_DIR` is a standard environment variable which should already be defined.

To add `GNOME_KEYRING_CONTROL` to your environment, define it to your `~/.bashrc`.

```
$ echo "export GNOME_KEYRING_CONTROL=${XDG_RUNTIME_DIR}/keyring/" >> ~/.bashrc
```

Run `start-keyring-daemon` whenever you need to unlock your keyring manually.

## Adding mail credentials to a keyring

Credentials can be added to gnome-keyring through a GUI program like [seahorse](https://wiki.gnome.org/Apps/Seahorse/) or with [secret-tools](http://manpages.ubuntu.com/manpages/bionic/man1/secret-tool.1.html) in a terminal. Credentials added to the keyring are associated with attributes which uniquely identify them. Network credentials must have the `server`, `port`, `user` and `protocol` attributes.

To add gmail credentials to an unlocked keyring:

```
$ sudo apt-get install libsecret-1-0 libsecret-1-dev libsecret-tools
$ secret-tool store --label='gmail send-email' \
            server smtp.googlemail.com \
            port 587 \
            protocol smtp \
            user my-user@gmail.com \
            xdg:schema org.gnome.keyring.NetworkPassword
```

This will prompt you for the secret to associate to those credentials. For gmail account using two factor authentication, an [app password](https://support.google.com/accounts/answer/185833?hl=en) is required instead of the account password.

Check if the credentials were added correctly using `secret-tool`:

```
$ secret-tool search protocol smtp

[/org/freedesktop/secrets/collection/login/20]
label = gmail send-email
secret = secretapppassword (in clear text, since keyring is now unlocked)
created = 2021-01-23 22:31:26
modified = 2021-01-23 22:31:26
schema = org.gnome.keyring.NetworkPassword
attribute.user = my-user@gmail.com
attribute.port = 587
attribute.server = smtp.googlemail.com
attribute.protocol = smtp
```

`secret-tool` returns hashed values when the keyring is locked. If it's unlocked, clear text values are outputted.

## Reading keyring credentials with git

When git performs an operation requiring credentials, it'll first check if a `credential helper` is defined. The credential helper fetches credentials on behalf of git. By itself, git doesn't know how to interact with gnome-keyring. Luckily a credential helper for gnome-keyring is available upstream. To activate it, use the following command:

```
git config --global credential.helper /usr/share/doc/git/contrib/credential/libsecret/git-credential-libsecret
```

Now, each time git looks for credentials, it'll first check your keyring for matching credentials 

`git credential` can be used to confirm access to the mail credentials:

```
$ echo "host=smtp.googlemail.com:587" >> /tmp/cred-request
$ echo "username=my-user@gmail.com" >> /tmp/cred-request
$ echo "protocol=smtp" >> /tmp/cred-request
$ cat /tmp/cred-request | git credential fill

protocol=smtp
host=smtp.googlemail.com:587
username=my-user@gmail.com
password=secretapppassword (in clear text)
```

## Using git send-email
To tell git send-email which credentials to use:

```
git config --global sendemail.smtpserver smtp.googlemail.com
git config --global sendemail.smtpencryption tls
git config --global sendemail.smtpserverport 587
git config --global sendemail.smtpuser my-user@gmail.com
```

Now jump to your git repository, and send yourself your last commit as an email patch:
```
git send-email -1 --to my-user@gmail.com
```

If everything went right, you're not prompted for a password prompt and the patch goes straight to your inbox.
