---
layout: post
title:  "Fix FIO (posixaio) issue on FreeBSD 10.3"
date:   2016-06-08 22:49:14 +0800
categories: FreeBSD
---
AIO module is not enabled in FreeBSD 10.3, as a result, when you run FIO with posixaio engine, it will complain `Function not implemented`. Another proof for AIO disabled on FreeBSD 10.3 is using sysctl to check aio options. You can use `sysctl -da|grep aio`. The default output is as follows:
{% highlight ruby %}
p1003_1b.aio_listio_max:
p1003_1b.aio_max:
p1003_1b.aio_prio_delta_max:
{% endhighlight %}

Please enable AIO by `kldload aio` and check whether it is loaded `kldstat`. If you find "aio" is displayed, then it was loaded. Double check the sysctl output, and you can see:
{% highlight ruby %}
kern.features.aio: Asynchronous I/O
vfs.aio.max_buf_aio: Maximum buf aio requests per process (stored in the process)
vfs.aio.max_aio_queue_per_proc: Maximum queued aio requests per process (stored in the process)
vfs.aio.max_aio_per_proc: Maximum active aio requests per process (stored in the process)
vfs.aio.unloadable: Allow unload of aio (not recommended)
vfs.aio.aiod_lifetime: Maximum lifetime for idle aiod
vfs.aio.aiod_timeout: Timeout value for synchronous aio operations
vfs.aio.num_buf_aio: Number of aio requests presently handled by the buf subsystem
vfs.aio.num_queue_count: Number of queued aio requests
vfs.aio.max_aio_queue: Maximum number of aio requests to queue, globally
vfs.aio.target_aio_procs: Preferred number of ready kernel threads for async IO
vfs.aio.num_aio_procs: Number of presently active kernel threads for async IO
vfs.aio.max_aio_procs: Maximum number of kernel threads to use for handling async IO
p1003_1b.aio_listio_max:
p1003_1b.aio_max:
p1003_1b.aio_prio_delta_max:
{% endhighlight %}
