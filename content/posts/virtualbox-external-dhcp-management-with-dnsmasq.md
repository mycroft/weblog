---
title: VirtualBox and external dhcp management with dnsmasq
date: 2013-04-18T13:28:00
summary: Tweaking VirtualBox to setup a dnsmasq as a dhcp server 
tags:
  - linux
---

I use [VirtualBox](https://www.virtualbox.org/) like many for my development VMs. One of mine problems with VirtualBox is, sometimes, that we want to manage their IPs in a more complex fashion than what's possible with its internal DHCP server (when using "host-only" interfaces), mostly because it doesn't allow statc IP configuration.

It's possible to disable this DHCP server and use another server, like [dnsmasq](http://www.thekelleys.org.uk/dnsmasq/doc.html). This DHCP server, which also is a DNS forwarder, is very light and can be installed on a developement server without bothering local network.

Configuration I use for my VMs is the following:

* Network/Adapter1: Attached to NAT;
* Network/Adapter2: Host-only Adapter; Name: vboxnet0

First step is to disable VirtualBox's DHCP server. You can do this in the VirtualBox GUI or using the command line:

```sh
# VBoxManage list dhcpservers
NetworkName:    HostInterfaceNetworking-vboxnet0
IP:             192.168.56.100
NetworkMask:    255.255.255.0
lowerIPAddress: 192.168.56.101
upperIPAddress: 192.168.56.254
Enabled:        Yes

# VBoxManage dhcpserver remove --netname HostInterfaceNetworking-vboxnet0

# VBoxManage list dhcpservers
#
```

You'll have to reboot all VMs to make sure this setting is considered. Unless this, the old DHCP server will still work and will provide an answer before dnsmasq.

Then, we can install dnsmasq, enable its DHCP server and only bind it on the VirtualBox's vboxnet0 network interface (that manages "host-only" adapters).

```sh
# apt-get install dnsmasq
[...]
```

And modify the */etc/dnsmasq.conf* configuration file:

```ini
interface=vboxnet0
dhcp-range=192.168.56.100,192.168.56.132
dhcp-host=08:00:27:5d:47:c9,192.168.56.133
```

Finally, start dnsmasq:

```sh
# service dnsmasq start
 * Starting DNS forwarder and DHCP server dnsmasq                        [ OK ] 
#
```

We'll check it works correcly by reading logs on the host and on the guest:
```sh
# tail -f /var/log/syslog
Jun 16 11:27:46 lambda dnsmasq-dhcp[16074]: DHCPDISCOVER(vboxnet0) 192.168.56.101 08:00:27:5d:47:c9 
Jun 16 11:27:46 lambda dnsmasq-dhcp[16074]: DHCPOFFER(vboxnet0) 192.168.56.133 08:00:27:5d:47:c9 
```

For more information about dnsmasq, check [Dnsmasq](https://help.ubuntu.com/community/Dnsmasq) page on Ubuntu's website.
