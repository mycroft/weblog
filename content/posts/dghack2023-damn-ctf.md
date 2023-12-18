---
title: "DG'hAck 2023: Damn CTF!"
date: 2023-12-01T11:58:41+01:00
summary: |
  Solving the DG'h4ck 2023 challenge: Damn Deprecation
tags:
- dghack2023
- ctf
---

I recently put myself into security again, and after having played a bit with [root-me](https://root-me.org/), we decided to take a serious look at the DG'hAck CTF. I found this year especially hard, way harder than in 2020 (or I'm getting older). There was a lot of different challenges, and this post sums-up them and how we solved them.

I wanted to write my resolution to the challenges, but as they went down, I could not reproduce nor do the correct level of writings I wished. So here is a sum-up with other people writeups instead.

Anyway, we (the `#olschool` team) finished first:

![DG'h4ck 2023 scoreboard](images/dghack-2023.png)

## Feed This Dragon

A simple javascript/json api game. It was purely code to write to automatize playing it (just click on the right elements, pay for upgrades, that kind of stuff), breaking multiple achievements and getting the flag at the end. [writeup](https://github.com/retyuil/DGHACK-2023-write_up/tree/main/Feed%20This%20Dragon)

## CryptoNeat

A web page with broken AES CTR key/nonce re-use. Once the [vulnerability](https://crypto.stackexchange.com/questions/2991/why-must-iv-key-pairs-not-be-reused-in-ctr-mode) (2 ciphers starting with the same block, ie IV) and one plaintext found, the second was really easy to decrypt. [writeup](https://nouman404.github.io/CTFs/DGHACK_2023/Crypto/Cryptoneat).

## TicToc

A classic in the CTF challenges. A password to guess, each good characters and the login takes 0.2 more seconds to validate than the other, retry until finding the password.

## Wrongsomewhere

A Windows ransomware to dissect to find out how to retrieve a key to decrypt files including the flag. [writeup](https://github.com/retyuil/DGHACK-2023-write_up/tree/main/Wrong%20somewhere), [writeup](https://eshard.com/posts/dghack2023-wrongsomwhere).

## Infinite Money Glitch

A website on which you win free money when viewing a video. Automatizing viewing and solving "captchas" (more like OCR with `cv2`) permited to grab the flag in a couple of minutes.

## Plugin Again

Basic web service with XSS/CSRF. [writeup](https://github.com/DarkInfern010/DGHack-2023---WriteUp/blob/main/pluginAgain/Readme.md); [writeup](https://nouman404.github.io/CTFs/DGHACK_2023/Web/Plugin_Again)

## Remove Before Flight

A simple web service, with some blind sql injection (sqlmap helped), then some little crypto to break password, and finally a token to retrieve thanks to a local file injection.

## Awesome Doc Converter

Another web service with a redis & wkhtmltopdf to exploit.

## AEgisSecureForge

We started with some pcap dump, to find out a privatebin-hosted document with source code, and it happened the source code was a service to exploit, to dump a private certificate. The crypto usage was wrong and the goal was to exploit bad practices. 

## PoliceForensic

An Android AVD snapshop to forensic. Actually we had some luck on this one has the challenge was broken and we found the flag a way that was fixed after. [writeup](https://github.com/0xNemo/CTF-challenges/blob/main/DGHACK_2023/PoliceForensic.md).

## KeepQuiet

We're a sysadmin and we had to exploit a web service to discover some .net malware.

## Damn Deprecation

A particular good one! You've a shell on a box with encrypted databases. You find out you can't do certain things on the box, as the kernel prevents to read its binary, nor moving it or such things. Even modifying `grub` was not possible. A strange `/README_NOW.txt` won't stop to pop.

It was easy to identify the kernel was patched with a rootkit or something. It was found out the kernel was installed from a given repository (that we don't have access). I found out that moving `/boot` was possible, so I recreated a whole `/boot`, booting on another kernel with some symlinks, that allowed me to get the hacked kernel binary.

I started to debug it locally using `qemu`, found out it was launching a `python` process (with its encrypted code on commandline). The code was quickly decrypted as its encryption key was set in the environment in the `kernel_execve` call. The `python` script launched python bytecode, and after some reversing, it was clear the process was calling home a distant daemon. Some `tcpdump` allowed to get the first flag (which was indeed the database encryption key).

The 2nd flag was found out by decryption the database.

Then, the last flag was still missing, and some reverse of the kernel began. As the `kernel_execve` call of the `python` process was new, I followed the code around the call, found out new functions that were not existing in the current kernel code, and discovered a rather big function I started to reverse, finding the last flag, generated in its process (after some tweaking to enter in some code that seemed disabled). [writeup](https://eshard.com/posts/dghack2023-damndeprecation).

## L'an 1, et puis l'an 2

We start with a Windows memory dump, and a pcap traces. We find out in those some Empire like malware, and have to decrypt its activity to find out the flags [writeup](https://blog.itarow.xyz/posts/l_an_1_et_puis_l_an_2/)

## Catch Him If You Can

A little scenario in which you play a sysadmin, and have to detect an attacker and answer some questions.

## My Virtual Bookstore

A windows binary to remotely exploit.

## A Maze In

As every year, a goodie in some event with a flag inside!

## Time to Fight Back

A bot running on a python-written javascript jail had to be exploited to gain RCE on attackers' system, then gain a shell by exploiting some docker misconfiguration, and finally some forensic to find the flags.

## Save the earth or loose yourself?!

Windows pentest. [writeup](https://github.com/nikaiw/dghack2023/blob/main/save-the-earth.md)

## Randigma

A custom Enigma machine with some entropy for the reflectors. I skipped this challenge. [writeup](https://github.com/Electro-Bug/DGHACK23/blob/main/Randigma/solve.md).

## JarJarBank

A java/spring application with 2 vulnerabilties: A custom SOAP request vulnerable to reading some files on the FS allowed to gather a token to reset/modify passwords; A second deserialization vulnerability allowed to popup a shell on the app's container. [writeup](https://github.com/nisharpe/write-ups/blob/main/dghack2023/JarJarBank/README.md).

## Android Mirrors

An Android application with 256 switches. Coded in kotlin, hard to correctly decompile, and a maze/algorithm to solve after having reversing it.

# Pixle

A ptrace/nanomite challenge, with a console image to rebuild. The binary would launch a process that would launch another one and so until 16 processes were created, and the whole thing would send data to each other using `ptrace(PTRACE_SETREGS/PTRACE_GETREGS)` calls, to reorder data in memory.

Once the protocol known (it was simple some 7 bytes block swaps, and the algorithm found)

# A Maritime Journey

Some basic OSINT, with some interesting real life data! [writeup](https://github.com/ArnaudFB/WriteUp-DGHack-2023/tree/main/AMaritimeJourney)

# TicTacPwn

An x64 binary to exploit. [writeup](https://github.com/Electro-Bug/DGHACK23/blob/main/tictacpwn/tictacpwn-solv.py); [writeup](https://github.com/TheHack42/dghack2023/tree/main/tictacpwn)

That's it folks.