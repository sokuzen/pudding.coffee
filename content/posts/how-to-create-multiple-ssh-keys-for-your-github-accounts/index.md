+++
title = "How to Create Multiple SSH Keys for each of your Github Accounts"
description = "Can I separate my work and personal Github accounts? Yes, you can set up multiple SSH keys on a single computer! Follow this guide to do exactly the same."
toc = true
authors = "pudding"
tags = ["git", "ssh", "web", "Github", "bash", "keys"]
categories = ["tech"]
series = []
date =  "2022-08-26T22:12:31+08:00"
lastmod = "2022-09-06T23:00:31+08:00"
featuredImage = "featured.jpg"
draft = false
+++

Photo by [Florian Berger](https://unsplash.com/@bergerteam?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText) on [Unsplash](https://unsplash.com/s/photos/shelves?utm_source=unsplash&utm_medium=referral&utm_content=creditCopyText)

## How do I separate my work and personal Github accounts?

Let's say you're working from home.

You are using your personal computer for your personal projects. Then you find your new employer is using Github. You already have your computer set up with an SSH key for your personal Github account, and [like some people suggest](https://www.quora.com/Should-I-use-separate-Github-accounts-for-personal-vs-professional-projects), you are creating a separate Github account for your almighty employer. 

You know, work-life balance?

You may have thought of using HTTPS, but you want to be one of the cool guys and type fewer passwords...

If you find yourself in that situation (or something similar), then you're in the right place! Just do the following, and you'll be cool! :nerd:

## Assumptions

I'm assuming that:
* you have git on your computer (thru Git Bash, or perhaps on your *nix-based terminal),
* you already have a Github account set up with an SSH key on the computer you will be using. To ensure if you already have an SSH key on your computer, type in (on Git Bash/terminal):
        
        $ ls ~/.ssh

The directory should exist and you should see key related files. If not, follow this [excellent guide](https://docs.github.com/en/authentication/connecting-to-github-with-ssh) instead from Github and go back here once done.

## 1. Generate a new SSH key

Assuming you already have Git Bash/terminal open, first, cd to the `.ssh` directory.

    $ cd ~/.ssh  

The `cd` saves us the pain of typing the directory we want to save the key. 

Then, paste in the following command, replacing your email:

    $ ssh-keygen -t ed25519 -C <your_email@example.com>

If that doesn't work, use the following command instead (you're likely using a system that doesn't support Ed25519, so we went with RSA):

    $ ssh-keygen -t rsa -C <your_email@example.com>

When prompted with `Enter a file in which to save the key`, enter a new filename. For this case, I put in `id_ed25519_work`:

    $ ssh-keygen -t ed25519 -C "pudding@pudding.coffee"
    Generating public/private ed25519 key pair.
    Enter file in which to save the key (/c/Users/you/.ssh/id_ed25519): id_ed25519_work

You should then be prompted with `Enter passphrase (empty for no passphrase):`. Feel free to hit enter/return if you don't like to add a passphrase, or add your phrase correspondingly.

If everything is good, you should see your key fingerprint generated and the key's randomart image. Typing in `ls` should show the new key file `id_ed25519_work` as well as its corresponding `.pub` file.

## 2. Configure hosts

Create/edit your ssh config file:

    $ vim ~/.ssh/config

Then add a host entry. Use the following template, replace the `Host` (`github.com-<whatever>`) and `IdentityFile` (your key file name generated from the last section) fields accordingly:

    Host github.com-work
        HostName github.com
        User git
        IdentityFile ~/.ssh/id_ed25519_work

Take note of the host you used, as this will be your new name that points to the new key. You will also see it used on the other sections

In case you're lost with vim, just type in `i` to enable insert/typing. Once everything is written, type in `Esc`, then `:wq`.

## 3. Add your new SSH key to your SSH agent

Before anything else, let's ensure the SSH agent is running. Paste in the following command:

    $ eval "$(ssh-agent -s)"
    
This should print the running agent's PID. Then add the private key to the SSH agent. If you created a different file name in step 2, replace `id_ed25519_work` with yours:
    
    $ ssh-add ~/.ssh/id_ed25519_work

## 4. Add the SSH key to your Github account

First, copy your key to the clipboard using this quick one-liner command. Replace the filename with yours:

    $ clip < ~/.ssh/id_ed25519_work.pub

Now go to your account on Github.com. In the upper right corner of the page, click your profile photo, then click Settings > SSH and GPG keys (Access).

Add a decriptive Title (e.g., Personal Laptop, My Server, Switch, Fridge A, etc.). Then paste your key into the corresponding field. 

Finally, click the button `Add SSH Key`.

## 5. Test (optional)

You should now be able to connect to Github, using a different SSH key per account. Run:

    $ ssh -T git@github.com

This should output your personal Github username, assuming your first SSH key is used for your personal Github, like the following:

    Hi (username)! You've successfully authenticated, but GitHub does not provide shell access.

Don't worry about the last phrase, Github is not supposed to provide you access.

Now to test connectivity to your work account, replace `github.com-work` with the host you used in step 2:

    $ ssh -T git@github.com-work

Replace `github.com-work` with the host you used in step 2. This should output the same template as above, replacing the username.

## 6. Clone using the new host, or change your remote URLs

Simply replace `github.com` on the SSH URL for the repositories you want to clone via your secondary/work SSH key. For example, normally when we want to clone `sokuzen/pudding.coffee.git`:

    $ git clone git@github.com:sokuzen/pudding.coffee.git

To use your work account, change `github.com` to the hostname you noted from step 2 `github.com-work`:

    $ git clone git@github.com-work:sokuzen/pudding.coffee.git

Then, to change the remote URLs of the repositories you already have:

    $ git remote set-url origin git@github.com-work:sokuzen/pudding.coffee.git

    $ git remote -v
    origin  git@github.com-work:sokuzen/pudding.coffee.git (fetch)
    origin  git@github.com-work:sokuzen/pudding.coffee.git (push)

Don't forget to change your local user.name and user.email configs:

    $ git config user.name "Pudding"
    $ git config user.email "pudding@pudding.coffee"

## Rinse and Repeat

From here you can add another SSH key from another account to the same computer. Just repeat steps 1-6.

I haven't tried this on Mac or other systems, as well as non-Github repositories (e.g., BitBucket, Gitlab, etc.) but I'm hoping the same steps apply.

And that's it! I hope you learned something here. Tweet me or something and let me know what you think, or if something is amiss!