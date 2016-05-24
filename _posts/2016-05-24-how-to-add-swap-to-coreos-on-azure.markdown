---
layout: post
title:  How to add swap to CoreOS on Azure?
date:   2016-05-24
author: Tadas Šubonis
tags:
- azure
- coreos
- swap
image: /img/logo-coreos.png
---

Just a little while ago we needed to add swap to our CoreOS VMs on Azure but
we didn't find a straightforward instructions how to do that.

Hopefully, it will come handy for people that are searching for this ;)

## Instructions
Open waagent config file:

```
sudo vim /usr/share/oem/waagent.conf
```

add (or uncomment) lines

```
ResourceDisk.Format=y
ResourceDisk.Filesystem=ext4
ResourceDisk.MountPoint=/mnt/resource
ResourceDisk.EnableSwap=y
ResourceDisk.SwapSizeMB=4096
```

and now restart waagent

```
sudo systemctl restart waagent
```

If you happened to follow [this](https://github.com/coreos/docs/issues/52) tutorial,
you gonna have an unstable system that's prone to freezing. Don't do that.


## Links
 * [Azure Linux User Agent Guide](https://azure.microsoft.com/en-us/documentation/articles/virtual-machines-linux-agent-user-guide/)
 * [WA Linux Agent on GitHub](https://github.com/Azure/WALinuxAgent)
