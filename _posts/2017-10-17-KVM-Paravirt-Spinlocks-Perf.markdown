---
layout: post
title:  "KVM Paravirt Spinlocks Challenges and Status"
date:   2017-10-17 13:05:14 +0800
categories: jekyll theme
tail: I'm a tail
---

{:toc}

## Background

Spinlocks on paravirtualization environment suffer from two known issues: [Lock Holder Preemption (LHP) and Lock Waiter Preemption (LWP)][LHP_LWP].This document tries to find the status of paravirtual (PV) spinlock on Linux kernel 4.12 for KVM, as well as the methodologies used in exploring spinlock performance. It targets to give some inspirations for further study in this field.

### Classic spinlock issues

Traditionally, spinlock is implemented as a test-and-set pattern. But it has two issues: [unfair and cache-line bouncing][UNFAIR_CACHE_BOUNCING]. Ticket-spinlock (which has been replaced by qspinlock in 2015) only resolved the unfair issue, but the second issue was resolved by qspinlock. On bare metal machine, the qspinlock demonstrates a huge benefit compared with ticket-spinlock. But what is the status on virtualization environment? Is there anything specific on virtualization environment to be considered?

### Spinlock challenges on virtual environment

The LHP and LWP are the typical issues on virtualization environment. Different virtualization technology provides different improvements, which was named spinlock enlighten. Meanwhile, CPU also tries to address this issue by Pause Loop Exiting (PLE), but it is inaccurate to detect spinlock's spin operation. As a result, PLE's optimization is good for some cases but bad for others.

How to mitigate the impact of LHP and LWP is still a hot topic recent years. Please read the papers published in 2016 [Opportunistic Spinlocks][OPPORTUNISTIC_SPINLOCK] and 2017 [I-Spinlock][SPINLOCK_PAPER1]. For popular virtualization products, for example, KVM and XEN, they have provided their PV spinlock respectively. This document will introduce the KVM's PV spinlock.

## KVM PV Spinlock

### qspinlock in kernel

qspinlock was introduced into Linux kernel in 2015. It gives good performance compared with ticket spinlock on bare metal machine. You can read this article [Queued spinlock][QUEUED_SPINLOCK] to understand its working mechanism.

In order to support PV spinlock, Linux kernel provides a "structure pv_lock_ops" to allow developers to easily integrate implementations for different virtual environment. In addition, Linux optimizes the function callback by back patching. For KVM's PV spinlock, please read the implementation in file arch/x86/kernel/kvm.c.

However, in para-virtual VM, qspinlock does not work as good as classic test-and-set. See the commit log:
{% highlight ruby %}
commit 2aa79af64263190eec610422b07f60e99a7d230a
Author: Peter Zijlstra (Intel) <peterz@infradead.org>
Date:   Fri Apr 24 14:56:36 2015 -0400

    locking/qspinlock: Revert to test-and-set on hypervisors

    When we detect a hypervisor (!paravirt, see qspinlock paravirt support
    patches), revert to a simple test-and-set lock to avoid the horrors
    of queue preemption.
{% endhighlight %}
If you want to setup VM through KVM, the spinlock is in test-and-set mode default. If you want to enable para-virtual qspinlock, please read this blog: [KVM Paravirt Spinlock][KVM_PARAVIRT_SPINLOCK]

### Performance evaluation methodologies

We want to fully understand the KVM PV spinlock's performance under non-contention and high contention environment. First, we have to select the benchmark. Generally, Linux Kernel Compilation (LKC) is a good benchmark and was referred in many papers. In our experiment, we also use LKC.

For the test scenario, we plan to compare the test-and-set spinlock and PV qspinlock's performance. We setup a single VM to evaluate the non-contention performance.

For high contention test, the ideal environment is setup two VMs. Each of them was assigned all CPU cores. So, if the bare metal machine has 28 cores, the two VM both have 28 CPU cores. Meanwhile, we also setup another kinds of contention test: run LKC on bare metal machine and 1 VM simultaneously.

Qspinlock demonstrates advantages on overcommitted LKC. Here overcommitted means the parallel thread number is much bigger than the CPU number. So, in our perf test, overcommitted is applied.

### Experiment & analysis

The CPU used in this test is E5-2660 v4 2.00GHZ, 2 CPU, every CPU has 14 cores. Hyperthreading is disabled.
Linux: Ubuntu 17.04, Linux kernel is 4.12.

#### High contention test

3 scenarios were tested. All of them use "make -j36" since CPU cores are 28 in bare metal machine. For each scenario, the LKC was run 10 times. We take the average and stddev.

- Bare metal machine's qspinlock performance. 100% CPU cores were used. We can see average time is 338.1s with very small standard deviation.

|Baseline of spinlock performance |Average(s)| Stddev |
|Baremetal (vmlinuz-4.12.0)       | 338.1    |0.5676  |
{:.mbtablestyle}

- Contention scenario: bare metal vs. KVM PV spinlock enlighten. Bare metal has 28 CPU, and KVM also has 28 CPU. We see bare metal is much better than KVM: bare metal has higher performance as well as small standard deviation.

|Contention: Baremetal vs. KVM enlighten spinlock |Average(s)|Stddev |
|                              Bare metal (28 CPU)| 447.2    | 13.88 |
|          KVM spinlock enlighten spinlock(28 CPU)| 955.1    | 195.99|
{:.mbtablestyle}

- Contention scenario: bare metal vs. KVM test-and-set spinlock. Bare metal has 28 CPU, and KVM also has 28 CPU. We see bare metal is still better than KVM. But if we compare it with KVM PV spinlock, the test-and-set is much better. This explains why KVM PV spinlock is disabled by default.

|Contention: Baremetal vs. KVM test-and-set       |Average(s)|Stddev|
|                   Bare metal (28 CPU)           | 593.5    | 21.03|
|          KVM spinlock test-and-set (28 CPU)     | 654.8    | 64.99|
{:.mbtablestyle}

- Contention scenario: KVM test-and-set spinlock vs. KVM spinlock enlighten.

|Contention: KVM test-and-set vs. KVM enlighten   |Average(s)|Stddev|
|          KVM spinlock test-and-set              | 498.9    | 17.31|
|         KVM spinlock enlighten                  | 801.6    |357.27|
{:.mbtablestyle}

#### Non-contention test

1 VM was setup, and there is qemu parameter used to enable or disable PV spinlock. The VM was assigned all CPU cores (28). We run 10 times of "time make -j80" for overcommitted mode. We saw that there is almost no difference.

| Parallel Thread No| 80      |  80     |   80    |    80   |     80  |
| enlighten         |6m21.984s|6m16.045s|6m16.645s|6m17.136s|6m17.165s|
|test-and-set       |6m20.179s|6m13.918s|6m14.288s|6m14.399s|6m14.539s|
{:.mbtablestyle}

### Summary

KVM's PV spinlock still needs to be improved, but currently there is no efficient mechanism for the VM to communicate with Hypervisor to avoid LHP and LWP.

[LHP_LWP]: http://dl.acm.org/citation.cfm?id=3064180&dl=ACM&coll=DL&CFID=984392879&CFTOKEN=64855677
[UNFAIR_CACHE_BOUNCING]: https://lwn.net/Articles/590243/
[KVM_PARAVIRT_SPINLOCK]: https://clovertrail.github.io/jekyll/theme/2017/09/12/KVM-Paravirt-Spinlocks.html
[KVM_Install]: https://help.ubuntu.com/community/KVM/Installation
[SPINLOCK_PAPER1]: https://dl.acm.org/citation.cfm?id=3064180
[OPPORTUNISTIC_SPINLOCK]: https://dl.acm.org/citation.cfm?id=2903271
[QUEUED_SPINLOCK]: http://blog.csdn.net/chenyu105/article/details/51892686
