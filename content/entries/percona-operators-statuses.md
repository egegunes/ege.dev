---
title: "Cluster Statuses in Percona Kubernetes Operators"
date: 2021-07-21T15:10:57+03:00
---

In Kubernetes, all resources have a status field separated from their spec. The
status field is an interface both for humans or applications to read the
perceived state of the resource.

When you deploy our Percona Kubernetes Operators –  Percona Operator for
MongoDB or Percona Operator for MySQL – in your Kubernetes cluster, you’re
creating a custom resource (CR for short) and it has its own status, too. Since
Kubernetes operators mimic the human operator and aim to have the required
expertise to run software in a Kubernetes cluster; the status of the custom
resources should be smart.

You can get cluster status with the commands below, or via (Kubernetes API) for
Percona Operator for MySQL:

```
% kubectl get pxc
NAME            ENDPOINT                                   STATUS   PXC   PROXYSQL   HAPROXY   AGE
lisette-18537   lisette-18537-haproxy.subjectivism-22940   ready    3                3         87m

% kubectl get pxc <cluster-name> -o jsonpath='{.status}'
{
  "backup": {
    "version": "8.0.23"
  },
  "conditions": [
    {
      "lastTransitionTime": "2021-07-12T13:13:46Z",
      "status": "True",
      "type": "initializing"
    }
  ],
  "haproxy": {
    "labelSelectorPath": "...",
    "ready": 3,
    "size": 3,
    "status": "ready"
  },
  "host": "lisette-18537-haproxy.subjectivism-22940",
  "logcollector": {
    "version": "1.8.0"
  },
  "observedGeneration": 2,
  "pmm": {
    "version": "2.12.0"
  },
  "proxysql": {},
  "pxc": {
    "image": "percona/percona-xtradb-cluster:8.0.22-13.1",
    "labelSelectorPath": "...",
    "ready": 2,
    "size": 3,
    "status": "initializing",
    "version": "8.0.22-13.1"
  },
  "ready": 5,
  "size": 6,
  "state": "initializing"
}
```

And for Percona Operator for MongoDB:

```
% kubectl get psmdb
NAME             ENDPOINT                                                     STATUS   AGE
cynodont-26997   cynodont-26997-mongos.subjectivism-22940.svc.cluster.local   ready    85m


% kubectl get psmdb <cluster-name> -o jsonpath='{.status}'
{
  "conditions": [
    {
      "lastTransitionTime": "2021-07-12T13:13:39Z",
      "status": "True",
      "type": "initializing"
    }
  ],
  "host": "cynodont-26997-mongos.subjectivism-22940.svc.cluster.local",
  "mongoImage": "percona/percona-server-mongodb:4.4.6-8",
  "mongoVersion": "4.4.6-8",
  "mongos": {
    "ready": 1,
    "size": 3,
    "status": "initializing"
  },
  "observedGeneration": 2,
  "ready": 3,
  "replsets": {
    "cfg": {
      "ready": 1,
      "size": 3,
      "status": "initializing"
    },
    "rs0": {
      "initialized": true,
      "ready": 2,
      "size": 3,
      "status": "initializing"
    }
  },
  "size": 6,
  "state": "initializing"
}
```

As you can see there are several fields in the output: conditions, cluster
size, number of ready cluster members, statuses and versions of different
components, and the “state”. In the following sections, we’ll take a look at
every possible value of the state field.

## Initializing

While the cluster is progressing to readiness, CR status is “initializing”. It
includes creating the cluster, scaling it up or down, and updating the CR that
triggers a rolling restart of pods (for instance updating Percona Operator for
MySQL memory limits).

Percona Operator for MongoDB also reconfigures the replica set config if
necessary (for instance it adds the new pods as members to replset or removes
terminated ones). Replica set in MongoDB is a set of servers that implements
replication and automatic failover. Although they have the same name, it’s
different from the Kubernetes replica set. While this configuration is
happening or if there is an unknown/unpredicted error during it, the status is
also “initializing”.

Since version 1.7.0, the Percona Operator for MySQL can handle full crash
recovery if necessary. If a pod waits for the recovery, the cluster status is
“initializing”.

## Ready

The operator keeps track of the status of each component in the cluster.
Percona Operator for MongoDB has the following components:

1. mongod StatefulSet
2. configsvr StatefulSet if sharding is enabled
3. mongos Deployment if sharding is enabled

Percona Operator for MySQL components:

1. PXC StatefulSet
2. HAProxy StatefulSet if enabled
3. ProxySQL StatefulSet if enabled

All components need to be in “ready” status for CR to be “ready”. If the number
of ready pods controlled by the stateful set reaches the desired number, the
operator marks the component as ready. The readiness of the pods is tracked by
Kubernetes using readiness probes for each container in the pod. For example,
for a Percona XtraDB Cluster container to be ready “wsrep_cluster_status” needs
to be “Primary” and “wsrep_local_state” should be “Synced” or “Donor”. For a
Percona Server for MongoDB container to be ready, accepting TCP connections on
27017 is enough.

But ready as the CR status means more than that. CR “ready” means the cluster
(Percona Server for MongoDB or Percona XtraDB Cluster) is up and running and
ready to receive traffic. So, even if all components are ready, the cluster
status can be “initializing”. In the Percona Operator for MongoDB, the replica
set needs to be initialized and its config up-to-date. Also, with the 1.9.0
release of both operators, the load balancer needs to be ready if the cluster
is exposed with `exposeType: LoadBalancer`.

## Stopping

Version 1.9.0 introduced two new statuses:

1. Stopping
2. Paused

Stopping means the cluster is paused or deleted and its pods are terminating right now.

If you run `kubectl delete psmdb <cluster-name>` or `kubectl delete pxc
<cluster-name>`` the resource can be deleted quickly without a chance to see
“stopping” status. If you had finalizers (for example
“delete-pxc-pods-in-order” in Percona Operator for MySQL) deletion will be
blocked until the finalizer list is exhausted and you can observe “stopping”
status.

## Paused

Once the cluster is paused and all pods are terminated, the CR status becomes “paused”.

To pause the cluster: `kubectl patch <psmdb|pxc> <cluster-name> --type=merge -p '{"spec": {"pause": true}}'`

Keep in mind, when the cluster is paused and exposeType is LoadBalancer – Load
balancers are still there and you continue to pay for them.

## Error

Before 1.9.0, “error” status could mean two different things:

1. An error occurred in the operator during the reconciliation of the CR
2. One or more pods in a component are not schedulable

With 1.9.0, the “error” status means only the operator errors. If there is an
unschedulable pod, the cluster’s status will be initializing. If the cluster is
stuck in initializing for too long, it’s better to check the operator logs to
investigate.

```
% kubectl logs <operator-pod-name>
...
{"level":"info","ts":1626095618.9982307,"logger":"controller_psmdb","msg":"Created a new mongo key","Request.Namespace":"subjectivism-22940","Request.Name":"cynodont-26997","KeyName":"cynodont-26997-mongodb-keyfile"}
{"level":"info","ts":1626095619.0032709,"logger":"controller_psmdb","msg":"Created a new mongo key","Request.Namespace":"subjectivism-22940","Request.Name":"cynodont-26997","KeyName":"cynodont-26997-mongodb-encryption-key"}
{"level":"info","ts":1626095687.3783236,"logger":"controller_psmdb","msg":"initiating replset","replset":"rs0","pod":"cynodont-26997-rs0-1"}
{"level":"info","ts":1626095694.020591,"logger":"controller_psmdb","msg":"replset was initialized","replset":"rs0","pod":"cynodont-26997-rs0-1"}
{"level":"error","ts":1626095694.622869,"logger":"controller_psmdb","msg":"failed to reconcile cluster","Request.Namespace":"subjectivism-22940","Request.Name":"cynodont-26997","replset":"rs0","error":"undefined state of the replset member cynodont-26997-rs0-0.cynodont-26997-rs0.subjectivism-22940.svc.cluster.local:27017: 6","errorVerbose":"undefined state of the replset member cynodont-26997-rs0-0.cynodont-26997-rs0.subjectivism-22940.svc.cluster.local:27017: 6\ngithub.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb.(*ReconcilePerconaServerMongoDB).reconcileCluster\n\t/go/src/github.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb/mgo.go:210\ngithub.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb.(*ReconcilePerconaServerMongoDB).Reconcile\n\t/go/src/github.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb/psmdb_controller.go:449\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:256\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:232\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).worker\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:211\nk8s.io/apimachinery/pkg/util/wait.JitterUntil.func1\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:152\nk8s.io/apimachinery/pkg/util/wait.JitterUntil\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:153\nk8s.io/apimachinery/pkg/util/wait.Until\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:88\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1371","stacktrace":"github.com/go-logr/zapr.(*zapLogger).Error\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/github.com/go-logr/zapr/zapr.go:128\ngithub.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb.(*ReconcilePerconaServerMongoDB).Reconcile\n\t/go/src/github.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb/psmdb_controller.go:451\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:256\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:232\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).worker\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:211\nk8s.io/apimachinery/pkg/util/wait.JitterUntil.func1\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:152\nk8s.io/apimachinery/pkg/util/wait.JitterUntil\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:153\nk8s.io/apimachinery/pkg/util/wait.Until\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:88"}
% kubectl logs <operator-pod-name>
 
...
{"level":"info","ts":1626095618.9982307,"logger":"controller_psmdb","msg":"Created a new mongo key","Request.Namespace":"subjectivism-22940","Request.Name":"cynodont-26997","KeyName":"cynodont-26997-mongodb-keyfile"}
{"level":"info","ts":1626095619.0032709,"logger":"controller_psmdb","msg":"Created a new mongo key","Request.Namespace":"subjectivism-22940","Request.Name":"cynodont-26997","KeyName":"cynodont-26997-mongodb-encryption-key"}
{"level":"info","ts":1626095687.3783236,"logger":"controller_psmdb","msg":"initiating replset","replset":"rs0","pod":"cynodont-26997-rs0-1"}
{"level":"info","ts":1626095694.020591,"logger":"controller_psmdb","msg":"replset was initialized","replset":"rs0","pod":"cynodont-26997-rs0-1"}
{"level":"error","ts":1626095694.622869,"logger":"controller_psmdb","msg":"failed to reconcile cluster","Request.Namespace":"subjectivism-22940","Request.Name":"cynodont-26997","replset":"rs0","error":"undefined state of the replset member cynodont-26997-rs0-0.cynodont-26997-rs0.subjectivism-22940.svc.cluster.local:27017: 6","errorVerbose":"undefined state of the replset member cynodont-26997-rs0-0.cynodont-26997-rs0.subjectivism-22940.svc.cluster.local:27017: 6\ngithub.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb.(*ReconcilePerconaServerMongoDB).reconcileCluster\n\t/go/src/github.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb/mgo.go:210\ngithub.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb.(*ReconcilePerconaServerMongoDB).Reconcile\n\t/go/src/github.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb/psmdb_controller.go:449\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:256\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:232\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).worker\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:211\nk8s.io/apimachinery/pkg/util/wait.JitterUntil.func1\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:152\nk8s.io/apimachinery/pkg/util/wait.JitterUntil\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:153\nk8s.io/apimachinery/pkg/util/wait.Until\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:88\nruntime.goexit\n\t/usr/local/go/src/runtime/asm_amd64.s:1371","stacktrace":"github.com/go-logr/zapr.(*zapLogger).Error\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/github.com/go-logr/zapr/zapr.go:128\ngithub.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb.(*ReconcilePerconaServerMongoDB).Reconcile\n\t/go/src/github.com/percona/percona-server-mongodb-operator/pkg/controller/perconaservermongodb/psmdb_controller.go:451\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).reconcileHandler\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:256\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).processNextWorkItem\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:232\nsigs.k8s.io/controller-runtime/pkg/internal/controller.(*Controller).worker\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/sigs.k8s.io/controller-runtime/pkg/internal/controller/controller.go:211\nk8s.io/apimachinery/pkg/util/wait.JitterUntil.func1\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:152\nk8s.io/apimachinery/pkg/util/wait.JitterUntil\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:153\nk8s.io/apimachinery/pkg/util/wait.Until\n\t/go/src/github.com/percona/percona-server-mongodb-operator/vendor/k8s.io/apimachinery/pkg/util/wait/wait.go:88"}
```

You can try new statuses in version 1.9.0 of both Percona Operator for MongoDB
and Percona Operator for MySQL. Percona Operator for MongoDB was released in
June and Percona Operator for MySQL is on the way.
