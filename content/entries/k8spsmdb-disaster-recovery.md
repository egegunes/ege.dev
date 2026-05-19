---
title: "Disaster Recovery for MongoDB on Kubernetes"
date: 2021-10-08T15:10:57+03:00
---

*This is a joint post with Sergey Pronin.*

As per the glossary, Disaster Recovery (DR) protocols are an organization’s
method of regaining access and functionality to its IT infrastructure in events
like a natural disaster, cyber attack, or even business disruptions related to
the COVID-19 pandemic. When we talk about data, storing backups on remote
servers is enough to pass DR compliance checks for some companies. But for
others, Recovery Time Objectives (RTO) and Recovery Point Objectives (RPO) are
extremely tight and require more than just a backup/restore procedure.

In this blog post, we are going to show you how to set up MongoDB on two
distant Kubernetes clusters with Percona Distribution for MongoDB Operator to
meet the toughest DR requirements.

### What to Expect

Here is what we are going to do:

1. Setup two Kubernetes clusters
2. Deploy Percona Distribution for MongoDB Operator on both of them. The
   Disaster Recovery site will run a MongoDB cluster in unmanaged mode.
3. We are going to simulate the failure and perform a failover to DR site

In the 1.10.0 version of the Operator, we have added the Technology Preview of
the new feature which enables users to deploy unmanaged MongoDB nodes and
connect them to existing Replica Sets.

![figure-0](/images/k8spsmdb-disaster-recovery/blog_mongodr_0.png)

### Set it All Up

We are not going to cover the configuration of the Kubernetes clusters, but in
our tests, we relied on two Google Kubernetes Engine (GKE) clusters deployed in
different regions.

### Prepare Main Site

We have shared all the resources for this blog post in this GitHub repo. As a
first step we are going to deploy the operator on the Main site:

```
$ kubectl apply -f bundle.yaml
```

Deploy the MongoDB managed cluster with `cr-main.yaml`:

```
$ kubectl apply -f cr-main.yaml
```

It is important to understand that we will need to expose ReplicaSet nodes
through a dedicated service. This includes Config Servers. This is required to
ensure that ReplicaSet nodes on Main and DR can reach each other. So it is like
a full mesh:

![figure-1](/images/k8spsmdb-disaster-recovery/blog_mongodr_1.png)

To get there, cr-main.yaml has the following changes:

```
spec:
  replsets:
  - rs0:
    expose:
      enabled: true
      exposeType: LoadBalancer
  sharding:
    configsvrReplSet:
      expose:
        enabled: true
        exposeType: LoadBalancer
```

We are using the LoadBalancer Kubernetes Service object as it is just simpler
for us, but there are other options – ClusterIP, NodePort. It is also possible
to utilize 3rd party tools like Submariner to implement a private connection.

If you have an already running MongoDB cluster in Kubernetes, you can expose
the ReplicaSets without downtime by changing these variables.

### Prepare Disaster Recovery Site

The configuration of the Disaster Recovery site could be broken down into the
following steps:

1. Copy the Secrets from the Main cluster.
    1. system users secrets
    2. SSL keys – both used for external connections and internal replication traffic
2. Tune Custom Resource:
    1. run nodes in unmanaged mode – Operator does not control replicaset configuration and secrets generation
    2. expose ReplicaSets (the same way we do it on the Main cluster)
    3. disable backups – backups can be only taken on the cluster managed by the Operator

### Copy the Secrets

System user’s credentials are stored by default in my-cluster-name-secrets
Secret object and defined in spec.secrets.users. Apply this secret in the DR
cluster with kubectl apply -f yaml-with-secrets. If you don’t have it in your
source code repository or if you rely on the Operator to generate it, you can
get the secret from Kubernetes itself, remove the unnecessary metadata and
apply.

On main execute:

```
$ kubectl get secret my-cluster-name-secrets -o yaml > my-cluster-secrets.yaml
```

Now remove the following lines from metadata:

```
annotations
creationTimestamp
resourceVersion
selfLink
uid
```

Save the file and apply it to the DR cluster.

The procedure to copy SSL keys is almost the same as for users. The difference
is the names of the Secret objects – they are usually called <CLUSTER_NAME>-ssl
and <CLUSTER_NAME>-ssl-internal. It is also possible to specify them in
secrets.ssl and secrets.sslInternal in the Custom Resource. Copy these two keys
from Main to DR and reference them in the CR.

### Tune Custom Resource

cr-replica.yaml will have the following changes:

```
  secrets:
    users: my-cluster-name-secrets
    ssl: replica-cluster-ssl
    sslInternal: replica-cluster-ssl-internal
 
  replsets:
  - name: rs0
    size: 3
    expose:
      enabled: true
      exposeType: LoadBalancer
 
  sharding:
    enabled: true
    configsvrReplSet:
      size: 3
      expose:
        enabled: true
        exposeType: LoadBalancer
 
  backup:
    enabled: false
```

Once the Custom Resource is applied, the services are going to be created. We
will need the IP addresses of each ReplicaSet node to configure the DR site.

```
$ kubectl get services
NAME                  TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)           AGE
replica-cluster-cfg-0    LoadBalancer   10.111.241.213   34.78.119.1       27017:31083/TCP   5m28s
replica-cluster-cfg-1    LoadBalancer   10.111.243.70    35.195.138.253    27017:31957/TCP   4m52s
replica-cluster-cfg-2    LoadBalancer   10.111.246.94    146.148.113.165   27017:30196/TCP   4m6s
...
replica-cluster-rs0-0    LoadBalancer   10.111.241.41    34.79.64.213      27017:31993/TCP   5m28s
replica-cluster-rs0-1    LoadBalancer   10.111.242.158   34.76.238.149     27017:32012/TCP   4m47s
replica-cluster-rs0-2    LoadBalancer   10.111.242.191   35.195.253.107    27017:31209/TCP   4m22s
```

### Add External Nodes to Main

At this step, we are going to add unmanaged nodes to the Replica Set on the
Main site. In cr-main.yaml we should add externalNodes under replsets.[] and
sharding.configsvrReplSet:

```
  replsets:
  - name: rs0
    externalNodes:
    - host: 34.79.64.213
      priority: 1
      votes: 1
    - host: 34.76.238.149
      priority: 1
      votes: 1
    - host: 35.195.253.107
      priority: 0
      votes: 0
 
  sharding:
    configsvrReplSet:
      externalNodes:
      - host: 34.78.119.1
        priority: 1
        votes: 1
      - host: 35.195.138.253
        priority: 1
        votes: 1
      - host: 146.148.113.165
        priority: 0
        votes: 0
```

Please note that we add three nodes, but only two are voters. We do this to
avoid split-brain situations and do not start the primary election if the DR
site is down or there is a network disruption between the Main and DR sites.

### Failover

Once all the configuration above is applied, the situation will look like this:

![figure-2](/images/k8spsmdb-disaster-recovery/blog_mongodr_2.png)

We have three voters in the main cluster and two voters in the replica cluster.
That means replica nodes won’t have the majority in case of main cluster
failure and they won’t be able to elect a new primary. Therefore we need to
step in and perform a manual failover.

Let’s kill the main cluster:

```
gcloud compute instances list \
    | grep my-main-gke-demo \
    | awk '{print $1}' \
    | xargs gcloud compute instances delete --zone europe-west3-b

gcloud container node-pools delete \
    --zone europe-west3-b \
    --cluster my-main-gke-demo \
    default-pool
```

I deleted the nodes and the node pool of the main Kubernetes cluster so now the cluster is in an unhealthy state. Let’s see what mongos on the DR site says when we try to read or write through it:

```
% ./psmdb-tester
2021/09/03 18:19:19 Successfully connected and pinged 34.141.3.189:27017
2021/09/03 18:19:40 read failed: (FailedToSatisfyReadPreference) Encountered non-retryable error during query :: caused by :: Could not find host matching read preference { mode: "primary" } for set cfg
2021/09/03 18:19:49 write failed: (FailedToSatisfyReadPreference) Could not find host matching read preference { mode: "primary" } for set cfg
```

![figure-3](/images/k8spsmdb-disaster-recovery/blog_mongodr_3.png)

Normally, we can only alter the replica set configuration from the primary node
but in this kind of situation where you don’t have a primary and only have a
few surviving members, MongoDB allows us to force the reconfiguration from any
alive member.

Let’s connect to one of the secondary nodes in the replica cluster and perform
the failover:

```
kubectl exec -it psmdb-client-7b9f978649-pjb2k -- mongo 'mongodb://clusterAdmin:<pass>@replica-cluster-rs0-0.replica.svc.cluster.local/admin?ssl=false'
...
rs0:SECONDARY> cfg = rs.config()
rs0:SECONDARY> cfg.members = [cfg.members[3], cfg.members[4], cfg.members[5]]
rs0:SECONDARY> rs.reconfig(cfg, {force: true})
```

Note that the indexes of surviving members may differ in your environment. You
should check rs.status() and rs.config() outputs first. The main idea is to
repopulate config members with only surviving members.

After the reconfiguration, the replica set will have just three members and two
of them will have votes and a majority. So, they’ll be able to select a new
primary. After performing the same process on the cfg replica set, we will be
able to read and write through mongos again:

```
% ./psmdb-tester
2021/09/03 18:41:48 Successfully connected and pinged 34.141.3.189:27017
2021/09/03 18:41:49 read succeed
2021/09/03 18:41:50 read succeed
2021/09/03 18:41:51 read succeed
2021/09/03 18:41:52 read succeed
2021/09/03 18:41:53 read succeed
2021/09/03 18:41:54 read succeed
2021/09/03 18:41:55 read succeed
2021/09/03 18:41:56 read succeed
2021/09/03 18:41:57 read succeed
2021/09/03 18:41:58 read succeed
2021/09/03 18:41:58 write succeed
```

Once the replica cluster has become the primary, you should reconfigure all
clients that connect to the old main cluster and point them to the DR site.
