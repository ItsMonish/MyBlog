+++
date = '2025-08-26T18:24:58+05:30'
title = 'Honeyfused: An intrusion detection script using FUSE filesystem'
description = 'A Writeup about my motivations and learning leading to the development of the project'
categories = ['Python', 'Experiment']
tags = ['writeup']
keywords = ['FUSE', 'Python', 'Honeyfiles', 'Intrusion Detection']
summary = 'A Writeup about my motivations and learning leading to the development of the project'
+++

## Introduction
First things first, what is Honeyfused? It's a fun little side project that I spent some time on. It is on my [github repo](https://github.com/ItsMonish/honeyfused) if anyone's interested.

Honeyfused is in a sense an intrusion detection application using honey-files. The program emulates filesystem mounted on directories and presents files that are actually not part of the original filesystem. Since the program emulates folders that usually contains secrets and API keys, intruders and infostealers may try to access it and possibly exfil the files. Since we control the filesystem, we can generate alerts whenever the file is accessed or modified. That is the gist of it. In this article you'll find why I did this, how I did this and things like that.

## How did I get here?
I very recently learnt that you can implement virtual filesystems, that is, filesystem that are not actually present on the disk, using FUSE bindings. All one had to do is override the operating system filesystem function calls to **mimic** an actual filesystem and you can literally pull show a filesystem out of nowhere.

Armed with the knowledge, I thought "why can't I just implement a decoy filesystem with decoy files and generate alerts whenever the files are accessed?". But this is basically honey-files with extra steps, that is, the virtual filesystem. Then I thought a honey-file is always the same, always there in the same place, and it is usually something like "passwords.txt" or "keys.csv". So what if I could dynamically generate new file contents in important places, places where applications usually keep their tokens, secrets and keys, places which are more believable and look like legit files to an attacker, places which an infostealer would automatically enumerate. And hence, I got started to find out.

## The filesystem implementation
There are a few libraries that provide libfuse bindings in Python. Namely [fusepy](https://github.com/fusepy/fusepy), [python-fuse](https://github.com/libfuse/python-fuse), [mfusepy](https://github.com/mxmlnkn/mfusepy) etc. I choose to work with `mfusepy` as it is one of the still maintained one. I couldn't find any documentation about the module. But lucky for me, they maintain a folder of [example implementations](https://github.com/mxmlnkn/mfusepy/tree/master/examples). One of those aligns closely with what I want, a pure [in-memory virtual filesystem implementation](https://github.com/mxmlnkn/mfusepy/blob/master/examples/memory.py). Even though it did not clearly specify what should be actually done, it did give me a gist of how to do it.

However the example script didn't quite work out-of-the-box for me. So I made some modifications and additions of my own to emulate an actual filesystem. At the end of it, I had a script to create a virtual filesystem that can contain and display files, create new files, delete existing ones, create a directoy and so on.

## What to generate?
Now that, I can mount an filesystem whereever I want, I need to generate files. Not just any files, files that appear to be legit, files that should be placed in places where popular applications places them. Then I spent some time researching in popular applications and their configurations. Nah, I'm kidding. I used ChatGPT to tell me how popular applications stored their configs and keys. At the end of it, I had a list of applications that seemed to be good candidates for the decoy files. I settled for Kubernetes's kubeconfig, AWS's credential and config file, Azure's credentials.json, GCP's default application config file, Terraform's credential file, Docker's config file and lastly Bitcoin's config and wallet files. Mostly developer centric, I know.

With an list of candidate application, I then again asked ChatGPT to give me a blueprint of configuration files and what kind of data should I use to fill it with. With that, I used Python's `string.template` to craft template of files and wrote generator functions to dynamically populate the templates. It took me some time to cover this aspect of the ground but it was fine.

## Putting things together
At this point, I have a script to mount a fake filesystem anywhere I want and generator functions that can give me realistic looking files that popular applications use. Only thing left is to put these things together. I used a YAML file (`config.yml`) as the configuration file for user. With that user can specify which applications can the script emulate and present decoys for. Because we don't want to present decoys for already present applications.

It took going a little back and forth on things, until I finally managed to put it together. So now my script (actually scripts), managed to generate decoy files for applications provided in `config.yml` and mount it in place I wanted.

## The logging
I'll admit, I haven't done a lot of logging stuff in Python. Since `mfusepy` already provided debug logs for the filesystem, I tried to implement logging that gives debug information for my script. So I read the documentation (yes I actually did that), and implemented it. And then I remembered that I had to log alerts as well, because that is the whole point of the application. But I didn't want to throw all the log at once. So I seperated logs from `myfusepy`, debug information from my script and the alerts I want to give out. Turns out it's surprisingly easy than it sounds.

## Conclusion
It took me over an week to put all these together. I mean I didn't spent all my time on this, but okay. So basically I got an Intrusion detection system out that puts fake files on your filesystem and then alerts you when someone accesses it. It is geared more towards developers who keep these kinda files in their storage. But at the end this is just a stupid experiment. So I hope no one uses it as an actual security measure. Still it was pretty fun convincing the system that a filesystem exists when it doesn't and generate configuration files for various applications.
