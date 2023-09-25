---
title:  "Setup & running systemd user services"
date:   "2018-05-12T11:26:20+0200"
tags:
  - linux
summary: "How to manage new services for users with systemd."
---

Something we want when it comes to run bitcoin's, litecoin's, ethereum's daemons, is to make sure they are always up & running, even after a reboot, and somewhat securely. To achieve this, I opted for a systemd user service.

Let's setup this for bitcoin together!

* Firstly, I created a user `coins` just for this use:

```shell
$ adduser -d /home/coins -s /bin/bash -u coins -g coins coins
$ id coins
uid=1001(coins) gid=1002(coins) groups=1002(coins)
```

* Then, prepared the user service. On my system, I have all my daemons binary in /opt, so don't be surprised by bitcoin's daemon is located at `/opt/bitcoin/bin/bitcoind`. The user service file will be better located in the `$HOME/.config/systemd/user` directory.

```shell
$ su - coins
$ mkdir -p .config/systemd/user
$ cat .config/systemd/user/bitcoin.service
[Unit]
Description=Bitcoin daemon
After=network.target

[Service]
ExecStart=/opt/bitcoin/bin/bitcoind -daemon -conf=%h/.bitcoin/bitcoin.conf -pid=/run/user/%U/bitcoind.pid
RuntimeDirectory=bitcoind
Type=forking
PIDFile=/run/user/%U/bitcoind.pid
Restart=on-failure

# Provide a private /tmp and /var/tmp.
PrivateTmp=true

# Mount /usr, /boot/ and /etc read-only for the process.
ProtectSystem=full

# Disallow the process and all of its children to gain
# new privileges through execve().
NoNewPrivileges=true

# Deny the creation of writable and executable memory mappings.
MemoryDenyWriteExecute=true

[Install]
WantedBy=default.target
```

* Once done, we have to register our daemon within `systemd`. But before that, as the `coins` user will be accessed using `su -` (and everybody knows that `su -` doesn't open a session, right?), we have to tweak the system a little: We need to make sure the system believes the user is always logged on (using the login)...

```shell
$ sudo loginctl enable-linger coins
$ ls -ltrad /run/user/$(id -u coins)
drwx------. 3 coins coins 80 May 12 12:02 /run/user/1001
```

... and add the two following variables inside .bashrc:

```
# User specific aliases and functions
export XDG_RUNTIME_DIR="/run/user/$UID"
export DBUS_SESSION_BUS_ADDRESS="unix:path=${XDG_RUNTIME_DIR}/bus"
```

As we are not creating a complete session (we should use `machinectl login` for this, but our user doesn't have a password), we need the environment to be aware of the dbus socket path. This is done using both `XDG_RUNTIME_DIR` & `DBUS_SESSION_BUS_ADDRESS` variables.

The result will be that the `coins` is able to run systemctl --user using `su -`:

```shell
$ id
uid=1001(coins) gid=1002(coins) groups=1002(coins) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
$ systemctl --user status
● shuttle.mkz.me
    State: running
     Jobs: 0 queued
   Failed: 0 units
    Since: Sat 2018-05-12 12:02:45 CEST; 2min 2s ago
   CGroup: /user.slice/user-1001.slice/user@1001.service
           └─init.scope
             ├─9557 /usr/lib/systemd/systemd --user
             └─9559 (sd-pam)
```

Registering our `bitcoin.service` and starting it:

```shell
$ systemctl --user enable bitcoin-testnet
Created symlink /home/coins/.config/systemd/user/default.target.wants/bitcoin-testnet.service → /home/coins/.config/systemd/user/bitcoin-testnet.service.
$ systemctl --user start bitcoin
$ systemctl --user status bitcoin
● bitcoin.service - Bitcoin daemon
   Loaded: loaded (/home/coins/.config/systemd/user/bitcoin.service; enabled; vendor preset: enabled)
   Active: active (running) since Sat 2018-05-12 12:06:28 CEST; 3s ago
  Process: 9769 ExecStart=/opt/bitcoin/bin/bitcoind -daemon -conf=/home/coins/.bitcoin/bitcoin.conf -pid=/run/user/1001/bitcoind.pid (code=exited
 Main PID: 9770 (bitcoind)
   CGroup: /user.slice/user-1001.slice/user@1001.service/bitcoin.service
           └─9770 /opt/bitcoin/bin/bitcoind -daemon -conf=/home/coins/.bitcoin/bitcoin.conf -pid=/run/user/1001/bitcoind.pid

May 12 12:06:28 shuttle.mkz.me systemd[9557]: Starting Bitcoin daemon...
May 12 12:06:28 shuttle.mkz.me systemd[9557]: Started Bitcoin daemon.
$ journalctl -q --user-unit bitcoin
May 12 12:06:28 shuttle.mkz.me systemd[9557]: Starting Bitcoin daemon...
May 12 12:06:28 shuttle.mkz.me systemd[9557]: Started Bitcoin daemon.
```

And this is it!
