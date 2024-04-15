---
layout: post
title: Wireless adapters on systemd-nspawn containers
date: 2024-01-15 11:47 -0500
categories: [linux, systemd, nspawn, containers, wireless, wifi, network]
tags: [linux, systemd, nspawn, containers, wireless, wifi, network]
author: edu4rdshl
image:
  path: /wifi_adapter.png
  alt: Wireless adapter
excerpt: Passing a wireless adapter to a systemd-nspawn container is not as easy as it seems, but it's possible.
---

#### Update 2
15/04/2024: the feature is already available on the latest systemd versions. You can use the `Interface=` option on the `.nspawn` configuration file to bind the wireless adapter to the container. I.e.:

```bash
$ cat /etc/systemd/nspawn/container.nspawn
[Exec]
Boot=true
PrivateUsers=true

[Network]
Interface=wlan0
```
and it should just work.

#### Update 1
25/01/2024: it's going to change with [this pull request](https://github.com/systemd/systemd/pull/30956) and should just work out of the box using the `Interface=` option on the `.nspawn` configuration file. I will update this post or create a new one when the feature is released.

## The problem
The last night I was trying to setup a systemd-nspawn container that will use a wireless adapter to perform some WiFi pentesting. I said to myself, this is going to be easy, I just need to bind the wireless adapter to the container and that's it. But I was wrong, when I tried to start the container with `Interface=wlan0` on my `.nspawn` configuration file, I got the following error:


```bash
Failed to move interface wlan0 to namespace: Invalid argument
```

So, I did a quick search on the internet and I found that this is a [known issue](https://github.com/systemd/systemd/issues/7873). As per Lennart words the problem is (or was):

> Last time I looked wifi devices were not compatible with network namespaces in the Linux kernel, and there's nothing we can do about that in nspawn, it's a kernel limitation. As soon as the kernel gets fixed in this regard it should start working in nspawn too without further changes.

## The solution

A quick read in the comments will drive you to the solution, basically you need to:

- Get the `phy*` interface name of your wireless adapter.
- Get the PID of the init (systemd) of the container.
- Set the `netns` of the `phy*` interface to the PID of the init (systemd) on the container.

Putting all together you will get something like this:

```bash
# Get the phy* interface name of your wireless adapter
$ iw dev
phy#0
    Interface wlan0
        ifindex 3
        wdev 0x1
        addr 00:11:22:33:44:55
        type managed
        channel 1 (2412 MHz), width: 20 MHz, center1: 2412 MHz
        txpower 20.00 dBm
```
then the `phy*` interface name is `phy0` - note the `#0` part change.

```bash
# Get the PID of the init (systemd) on the container
$ machinectl status ContainerName | awk '/Leader:/{print $2}'
1234
```
then the PID of the init (systemd) on the container is `1234`.

```bash
# Set the netns of the phy* interface to the PID of the init (systemd) on the container
$ sudo iw phy phy0 set netns 1234
```

Now you can get a shell to the systemd-nspawn container and it will have access to the wireless adapter.

```bash
$ machinectl shell ContainerName
$ ip link
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN mode DEFAULT group default qlen 1000 link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
3: wlan0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP mode DORMANT group default qlen 1000 link/ether 00:11:22:33:44:55 brd ff:ff:ff:ff:ff:ff
```
and now you can use the wireless adapter on the container as you wish.

## References

- https://github.com/systemd/systemd/issues/7873

The next post will be about how to get container's GUI apps running on a Wayland session (GNOME) using a nested `mutter` instance. Stay tuned!