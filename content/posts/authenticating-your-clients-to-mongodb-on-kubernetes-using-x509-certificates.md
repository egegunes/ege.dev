---
title: "Authenticating Your Clients to Mongodb on Kubernetes Using X509 Certificates"
date: 2022-02-13T20:42:57+03:00
showDate: true
draft: false
---

Managing database users and their passwords can be a hassle. Sometimes, they
could even wait in various configuration files, hardcoded. Using certificates
can help you avoid the toil of managing, rotating, and securing user passwords,
so let’s see how to have x509 certificate authentication with the [Percona
Server for MongoDB
Operator](https://www.percona.com/doc/kubernetes-operator-for-psmongodb/index.html)
and cert-manager.

[cert-manager](https://cert-manager.io/) is our recommended way to manage TLS
certificates on Kubernetes clusters. The operator is already integrated with it
to generate certificates for TLS and cluster member authentication. We’re going
to leverage cert-manager APIs to generate valid certificates for MongoDB
clients.

There are rules to follow to have a valid certificate for user authentication:

1. A single Certificate Authority (CA) MUST sign all certificates.
2. The certificate’s subject MUST be unique.
3. The certificate MUST not be expired.

For the complete requirements, check the [MongoDB
docs](https://docs.mongodb.com/manual/core/security-x.509/#client-certificate-requirements).

## Creating Valid Certificates for Clients

Let’s check our current certificates:

```shell
$ kubectl get cert
NAME                      READY   SECRET                    AGE
cluster1-ssl              True    cluster1-ssl              17h
cluster1-ssl-internal     True    cluster1-ssl-internal     17h
```

The operator configures MongoDB nodes to use "cluster1-ssl-internal" as the
certificate authority. We’re going to use it to sign the client certificates to
conform to Rule 1.

First, we need to create an Issuer:

```shell
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
 name: cluster1-psmdb-x509-ca
spec:
 ca:
   secretName: cluster1-ssl-internal
EOF
```

Then, our certificate:

```shell
$ kubectl apply -f - <<EOF
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
 name: cluster1-psmdb-egegunes
spec:
 secretName: cluster1-psmdb-egegunes
 isCA: false
 commonName: egegunes
 subject:
   organizations:
     - percona
   organizationalUnits:
     - cloud
 usages:
   - digital signature
   - client auth
 issuerRef:
   name: cluster1-psmdb-x509-ca
   kind: Issuer
   group: cert-manager.io
EOF
```

The "usages" field is important. You shouldn’t touch its values. You can change
the "subject" and "commonName" fields as you wish. They’re going to construct
the Distinguished Name (DN) and DN will be the username.

```shell
$ kubectl get secret cluster1-psmdb-egegunes -o yaml \
    | yq3 r - 'data."tls.crt"' \
    | base64 -d \
    | openssl x509 -subject -noout

subject=O = percona, OU = cloud, CN = egegunes
```

Let's create the user:

```javascript
rs0:PRIMARY> db.getSiblingDB("$external").runCommand(
 {
   createUser: "CN=egegunes,OU=cloud,O=percona",
   roles: [{ role: 'readWrite', db: 'test' }]
 }
)

{
       "ok" : 1,
       "$clusterTime" : {
               "clusterTime" : Timestamp(1643099623, 3),
               "signature" : {
                       "hash" : BinData(0,"EdPrmPJqfgRpMEZwGMeKNLdCe10="),
                       "keyId" : NumberLong("7056790236952526853")
               }
       },
       "operationTime" : Timestamp(1643099623, 3)
}
```

We’re creating the user in the "$external" database. You need to use
"$external" as your authentication source. Note that we’re reversing the
subject fields, this is important.

## Authenticating With the Certificate

I have created a simple Go application to show how you can use x509
certificates to authenticate. It’s redacted here for brevity:

```go
// ca.crt is mounted from secret/cluster1-ssl
caFilePath := "/etc/mongodb-ssl/ca.crt"

// tls.pem consists of tls.key and tls.crt, they're mounted from
secret/cluster1-psmdb-egegunes
certKeyFilePath := "/tmp/tls.pem"

endpoint := "cluster1-rs0.psmdb.svc.cluster.local"

uri := fmt.Sprintf(
       "mongodb+srv://%s/?tlsCAFile=%s&tlsCertificateKeyFile=%s",
       endpoint,
       caFilePath,
       certKeyFilePath,
)

credential := options.Credential{
       AuthMechanism: "MONGODB-X509",
       AuthSource:    "$external",
}

opts := options.Client().SetAuth(credential).ApplyURI(uri)

client, _ := mongo.Connect(ctx, opts)
```

The important part is using "MONGODB-X509" as the authentication mechanism. We
also need to pass the CA and client certificate in the MongoDB URI.

```shell
$ kubectl logs psmdb-x509-tester-688c989567-rmgxv
2022/01/25 07:50:09 Connecting to database
2022/01/25 07:50:09 URI: mongodb+srv://cluster1-rs0.psmdb.svc.cluster.local/?tlsCAFile=/etc/mongodb-ssl/ca.crt&tlsCertificateKeyFile=/tmp/tls.pem
2022/01/25 07:50:09 Username: O=percona,OU=cloud,CN=egegunes
2022/01/25 07:50:09 Connected to database
2022/01/25 07:50:09 Successful ping
```

You can see the complete example in [this
repository](https://github.com/egegunes/psmdb-x509-tester). If you have any
questions, please add a comment or create a topic in the [Percona
Forums](http://forums.percona.com/).
