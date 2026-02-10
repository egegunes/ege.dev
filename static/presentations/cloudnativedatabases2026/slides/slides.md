### Troubleshooting Databases on Kubernetes
<p style="font-size: 0.8em;">Cloud Native Databases - 2026-02-20</p>
<br>
<hr>
<p>Ege Güneş</p>
<p style="font-size: 0.7em;">Senior Software Engineer</p>

---
### About me

<img src="img/ege.jpeg" style="width: 200px; background:none; border:none; box-shadow:none;">

- Living in Istanbul, Turkey
- Working on [Percona Operators](https://www.percona.com/software/percona-operators) for ~5 years

---
### Premeditatio Malorum

<div class="r-fit-text">

“_What is quite unlooked for is more crushing in its effect, and unexpectedness
adds to the weight of a disaster. This is a reason for ensuring that nothing
ever takes us by surprise. We should project our thoughts ahead of us at every
turn and have in mind every possible eventuality instead of only the usual
course of events…_

**_Rehearse them in your mind: exile, torture, war, shipwreck. All the terms of
our human lot should be before our eyes._**_”_ — Seneca

</div>

---
### Why troubleshooting is harder on Kubernetes?

--

Images are (usually) rootless and containers are immutable.

--

Tools we use for debugging are not usable out of the box.

--

Reconciliation loops make tweaking stuff harder.

---
### Basic troubleshooting

--

#### Check logs (duh).

--

Pro-tip: Use a tool like [stern](https://github.com/stern/stern) to tail logs from multiple pods.
<pre><code data-trim data-noescape class="language-plaintext">
$ stern instance1 -c database --tail 1
cluster1-instance1-9fqb-0 database 2026-02-09 09:30:59,964 INFO: no action. I am (cluster1-instance1-9fqb-0), the leader with the lock
cluster1-instance1-4c5g-0 database 2026-02-09 09:30:59,974 INFO: no action. I am (cluster1-instance1-4c5g-0), a secondary, and following a leader (cluster1-instance1-9fqb-0)
cluster1-instance1-52h8-0 database 2026-02-09 09:30:59,974 INFO: no action. I am (cluster1-instance1-52h8-0), a secondary, and following a leader (cluster1-instance1-9fqb-0)
</code></pre>

--

#### Check events

--

Keep an eye on:

- CrashLoopBackOff
- OOMKilled
- FailedScheduling
- FailedAttachVolume
- FailedMount
- NodeNotReady
- DiskPressure
- MemoryPressure

--

Events are persisted only for **1 hour**!

--

#### Check statuses

--

Conditions usually contain useful information.
<pre><code data-trim data-noescape class="language-yaml" data-line-numbers="1-6|3">
- lastTransitionTime: "2026-02-09T07:15:10Z"
  message: pgBackRest replica create repo is not ready for backups
  observedGeneration: 1
  reason: StanzaNotCreated
  status: "False"
  type: PGBackRestReplicaRepoReady
</code></pre>

--

Pro-tip: Use [kubectl status](https://github.com/bergerx/kubectl-status#kubectl-status) plugin.

<img src="img/kubectl-status.png" class="r-stretch" style="background:none; border:none; box-shadow:none;">

---
### Sometimes you need to take control!

--

Bypass the StatefulSet lock.
<pre><code data-trim data-noescape class="language-plaintext">
NAME                           READY   STATUS    RESTARTS   AGE
cluster1-rs0-0                 4/4     Running   0          111m
cluster1-rs0-1                 3/4     Running   0          110m
</code></pre>
<pre><code data-trim data-noescape class="language-bash">
$ kubectl exec cluster1-rs0-1 -- touch /data/db/sleep-forever
</code></pre>
<p style="font-size: 0.5em;">MySQL: /var/lib/mysql/sleep-forever, PostgreSQL: /pgdata/sleep-forever</p>
<pre><code data-trim data-noescape class="language-plaintext">
NAME                           READY   STATUS    RESTARTS   AGE
cluster1-rs0-0                 4/4     Running   0          113m
cluster1-rs0-1                 4/4     Running   0          112m
cluster1-rs0-2                 4/4     Running   0          110m
</code></pre>

--

Stop the reconciliation.

<pre><code data-trim data-noescape class="language-bash">
$ kubectl patch pg cluster1 \
    --type=merge \
    -p '{ "spec": { "unmanaged": true } }'
</code></pre>

or if the operator doesn't have this kind of feature:

<pre><code data-trim data-noescape class="language-bash">
$ kubectl scale deployment percona-server-mongodb-operator \
    --replicas=0
</code></pre>

---
### Use your toolbox.

--

Ephemeral containers to the rescue.
<p style="font-size: 0.5em;">Since Kubernetes v1.25</p>

<pre><code data-trim data-noescape class="language-bash" data-line-numbers="1-5|3|5">
$ kubectl debug pod/cluster1-pxc-1 -it \
    --target=pxc \
    --share-processes=true \
    --image=egegunes/troubleshoot:ubi9 \
    --profile=sysadmin
</code></pre>

--

You can debug a node as well.

<pre><code data-trim data-noescape class="language-bash">
$ kubectl debug node/worker-node-0 -it \
    --image=egegunes/troubleshoot:ubi9 \
    --profile=sysadmin
</code></pre>

<p style="font-size: 0.7em;">The root filesystem of the Node will be mounted at /host.</p>

--

With debug containers you can run:
- [`percona-toolkit`](https://docs.percona.com/percona-toolkit/index.html) collection,
- `strace`,
- `tcpdump`,
- `iostat` and more.

--

You can go crazier.

<pre><code data-trim data-noescape class="language-bash">
$ kubectl exec -it cluster1-pxc-0 -- bash
bash-5.1$ vim
bash: vim: command not found
</code></pre>

No vim?? Let's install it!

--

<pre><code data-trim data-noescape class="language-bash">
$ kubectl debug -it cluster1-pxc-0 \
    --target=pxc \
    --share-processes=true \
    --image=registry.access.redhat.com/ubi9/ubi \
    --profile=sysadmin

$ yum install --downloadonly --downloaddir=/tmp/rpm vim
$ cp /tmp/rpm/* /proc/1/root/tmp/
$ nsenter --mount --net --pid --target 1 bash
$ cd /tmp/
$ rpm -Uvh --nodeps *.rpm
</code></pre>

--

Voila!

<pre><code data-trim data-noescape class="language-bash">
$ kubectl exec -it cluster1-pxc-0 -- which vim
/usr/bin/vim
</code></pre>

---
### Epilogue

Experimenting with these beforehand would help you be prepared when things break.

**Things will eventually break.**

---
### Percona Live is back!

<img src="img/perconalive.jpeg" style="background:none; border:none; box-shadow:none;">

---
### Reach out to us!

- Create topics on [Percona Community Forum](https://forums.percona.com/),
- Creates issues on [GitHub](https://github.com/percona),
- Contribute code!

---
## Thank you for listening!
