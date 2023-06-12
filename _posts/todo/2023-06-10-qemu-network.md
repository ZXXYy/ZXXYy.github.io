---
title:      "VM中的网络连接--以qemu为例"
date:       2023-06-10 09:00:00
author:     zxy
math: true
categories: ["Coding", "Network"]
tags: ["vm", "network"]
post: true
---

There are two parts to networking within QEMU:

- the virtual network device that is provided to the guest (e.g. a PCI network card).
- the network backend that interacts with the emulated NIC (e.g. puts packets onto the host's network).

By default QEMU will create a SLiRP user network backend and an appropriate virtual network device for the guest (eg an E1000 PCI card for most x86 PC guests), as if you had typed `-net nic -net user` on your command line.
## Reference
1. [QEMU Documentation/Networking](https://wiki.qemu.org/Documentation/Networking#Network_Basics)