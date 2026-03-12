---
title: "gremlins in digitalocean"
date: 2026-03-12
---

We are in the process of certifying our operators for DigitalOcean Kubernetes
Service (DOKS). We started with PostgreSQL Operator and it worked just fine
without any adjustments. Then we moved on to our MySQL Operators and it
surfaced a problem in HAProxy.

HAProxy is used by default in our MySQL clusters. They sit in front of MySQL
instances as the proxy to have read/write splitting. We have our own external
scripts to perform checks for each backend for determining if a MySQL server is
good for that particular backend.

After deploying the operator on DOKS, we realized that our HAProxy pods are
failing to get ready because all external checks are failing due to timeouts.
But why?

It's gotta be the DNS. It's always DNS, isn't it? Turns out, no. I tested DNS
queries from the HAProxy container and it seems they were fast.

Could this be AppArmor? Maybe DOKS has stricter AppArmor profiles? I configured
HAProxy pods to be `Unconfined`. It didn't help either.

Then I decided to increase the timeout from 10 seconds to 30 seconds, just to
see how much time the script needs to finish. To my surprise, the script was
taking 810 milliseconds to finish! How could a check time out in 10 seconds but
finish in less than a second with 30 seconds timeout?  

In every deep debugging session, there is a moment when engineers start to
believe in spiritual beings or gremlins who mess with the software in the
system. One needs to resist this temptation of irrationality. In computers
there's always a rational explanation for problems, it's just very deep and
caused by unlucky combinations of design choices and/or bugs.

At this point I decided to attach a debug container into the pod and check what
HAProxy is doing with strace. This resulted in the first breakthrough of the
problem: the child process of HAProxy that runs the external check command was
doing shitloads of poll syscalls until it was killed due to timeout. It wasn't
even running the script, it was simply stuck.

This realization made me shift my focus to HAProxy itself. I started to read
the source code to see what's going on.

[src/extcheck.c](https://github.com/haproxy/haproxy/blob/v2.8.0/src/extcheck.c#L414-L427)
```
int fd;
sa_family_t family;

/* close all FDs. Keep stdin/stdout/stderr in verbose mode */
fd = (global.mode & (MODE_QUIET|MODE_VERBOSE)) == MODE_QUIET ? 0 : 3;

my_closefrom(fd);

/* restore the initial FD limits */
limit.rlim_cur = rlim_fd_cur_at_boot;
limit.rlim_max = rlim_fd_max_at_boot;
if (raise_rlim_nofile(NULL, &limit) != 0) {
	getrlimit(RLIMIT_NOFILE, &limit);
	ha_warning("External check: failed to restore initial FD limits (cur=%u max=%u), using cur=%u max=%u\n",
		   rlim_fd_cur_at_boot, rlim_fd_max_at_boot,
		   (unsigned int)limit.rlim_cur, (unsigned int)limit.rlim_max);
}
```

HAProxy attempts to close all FDs. What does *all* mean? And then it restores
limits to some value? What's that value?

All FDs means closing all from FD 0 to FD soft limit. `my_closefrom` function
first polls the FD and then closes it if poll doesn't fail with `POLLNVAL`. OK,
this explains the excessive polling I see with strace. But it's the default
HAProxy behavior for a long time, why didn't we see the same problem on some
other platform, i.e GKE?

The answer is simple. Turns out, DOKS has a much higher soft limit than GKE. On
GKE `ulimit -n` returns 1048576, on DOKS 1073741816! But still there was
something that troubled me at this point: I vaguely remembered a configuration
option in HAProxy that limits the number of FDs that the process will use. The
option is `fd-hard-limit` and
[docs](https://docs.haproxy.org/2.8/configuration.html#fd-hard-limit) say its
default value is 1048576. If it had any effect, I wouldn't have seen any
problems. Something was wrong in HAProxy.

Remember that the external check process was doing something to restore limits
to some value assigned at boot? Those values are assigned in the `main`
function here:

[src/haproxy.c](https://github.com/haproxy/haproxy/blob/v2.8.0/src/haproxy.c#L3263-L3269)
```
/* take a copy of initial limits before we possibly change them */
getrlimit(RLIMIT_NOFILE, &limit);

if (limit.rlim_max == RLIM_INFINITY)
	limit.rlim_max = limit.rlim_cur;
rlim_fd_cur_at_boot = limit.rlim_cur;
rlim_fd_max_at_boot = limit.rlim_max;
```

[The
code](https://github.com/haproxy/haproxy/blob/v2.8.0/src/haproxy.c#L3311-L3312)
that limits FDs with `fd-hard-limit` is a few lines below:

```
if (global.fd_hard_limit && limit.rlim_cur > global.fd_hard_limit)
	limit.rlim_cur = global.fd_hard_limit;
```

This means external checks restore limits to the value unbounded by
`fd-hard-limit`. Oh, this looks like [a
bug](https://github.com/haproxy/haproxy/issues/3299) in HAProxy and explains
why we had this issue!
