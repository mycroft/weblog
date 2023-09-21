---
title: "ssh: Cleaning known_hosts obsolete entries"
date: 2023-09-21T20:36:54+02:00
draft: true
categories:
- ssh
summary: |
  ssh: Disable strict host checking or removing them
---

One thing that bothers me a lot when using ssh with VMs in my homelab is that I re-use IPs after destroying/recreating VMs. My `.ssh/known_hosts` then grows and contains a lot of obsolete entries, and eventually, it ends up conflicting.

Here the trick to disable strict host checking on connect for a subset of IPs. Just add the following in `.ssh/config`:

```sh
$ cat ~/.ssh/config
...
Host 10.2.1.*
  StrictHostKeyChecking no
  UserKnownHostsFile=/dev/null
...
```

Another possiblity is to remove them. Simply use `ssh-keygen -R`:

```sh
$ ssh-keygen -R "10.2.1.22"
# Host 10.2.1.22 found: line 40
# Host 10.2.1.22 found: line 41
# Host 10.2.1.22 found: line 42
/home/mycroft/.ssh/known_hosts updated.
Original contents retained as /home/mycroft/.ssh/known_hosts.old
```


