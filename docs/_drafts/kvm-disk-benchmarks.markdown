---
layout: post
title:  "Proxmox: A shallow dive into disk performance overheads with KVM, VirtIO and ZFS"
categories:
    - homelab
---

Today I'm writing about disk performance. More specifically, SSD performance in KVM (Kernel-based virtual machines) 
nodes using VirtIO disks on top of a ZFS filesystem. I'm not looking to get a deep understanding of this technology 
right now, and you, my dear reader, should not use what I've found here as a definitive answer. It may however serve as
a jumping off point. 

The reason why this benchmark exists is somewhat contrived. I want to run a proper HA Kubernetes cluster on my homelab.
This in and of itself is not a big issue, especially not with low-overhead solutions like [k3s](https://k3s.io/) that 
are well suited to a home environment with workloads that are, let's be honest, not that impressive. However, since I 
want this to be proper HA so that I can play with scaling and resilience testing, I need persistent data to be available
across all the nodes in the cluster. This introduces the need for a distributed block storage solution. In many such 
cases, and probably in most production environments, this problem is solved by using solutions like SANs or maybe
something hyper converged with Ceph or similar. My homelab is pretty small scale (for now...) and I'm looking for a 
more software and Kubernetes-oriented solution.

Enter: [Longhorn](https://longhorn.io/) (Not the Windows Vista prototype, I'm old). This seems like a suitable solution
for my use case, and it allows me to ensure that all persistent data is replicated across most/all of my nodes. However,
it is clear that there is a [performance penalty](https://github.com/longhorn/longhorn/issues/3037) (as with most things)
and I would like to get a feel for what the actual performance of my nodes will be. 

Let's get cracking!

## Hardware
* List of hardware stuff

## Benchmarks

* Link to ars - 

## Test setup

* Nodes and stuff

## Results

### Graph

![IOPS](/assets/kvm-disk-benchmarks/iops.svg)
![I/O Bandwidth](/assets/kvm-disk-benchmarks/io_bandwidth.svg)
![I/O Written](/assets/kvm-disk-benchmarks/io_written.svg)