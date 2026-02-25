---
title: "Encrypt PostgreSQL Data at Rest on Kubernetes"
date: 2025-02-07T15:10:57+03:00
---

The upcoming Percona Operator for PostgreSQL v2.6.0 release introduces support
for PostgreSQL 17, which opens exciting possibilities for data security. Since
[pg_tde](https://github.com/percona/pg_tde) comes pre-installed in Percona’s
official PostgreSQL 17 images, this release presents an excellent opportunity
to implement Transparent Data Encryption in your Kubernetes-deployed databases.
Let’s look at how to configure and use pg_tde with the Percona PostgreSQL
Operator.

### Understanding pg_tde

Transparent Data Encryption (TDE) offers encryption at the table level and
solves the problem of protecting data at rest. The Percona Distribution for
PostgreSQL currently offers the only open source implementation of TDE for
PostgreSQL. The encryption is transparent for users, allowing them to access
and manipulate the data without requiring application modifications.

While pg_tde has a build available for PostgreSQL Community Server, the
extension leverages extended APIs introduced in Percona Server for PostgreSQL
to provide more complete and performant encryption. You can read more about the
pg_tde extension in the [TDE beta announcement blog
post](https://www.percona.com/blog/open-source-postgresql-pg-tde-beta/).

The new Percona Operator release will include pg_tde pre-installed with
PostgreSQL 17, simplifying the implementation of encryption at rest. Note:
Currently, pg_tde is in the beta phase. Please try it out on non-production
setups and [share your feedback](https://github.com/percona/pg_tde/discussions/151)!

### Initial configuration

To begin, we need to configure PostgreSQL to load pg_tde during startup. This
configuration is managed through the deploy/cr.yaml file:

```
patroni:
  dynamicConfiguration:
    postgresql:
      parameters:
        shared_preload_libraries: pg_tde
```

After updating the configuration, apply it and restart your PostgreSQL pods:

```
kubectl apply -f deploy/cr.yaml

for sts in $(kubectl get statefulset -o name | grep cluster1-instance1); do
    kubectl rollout restart $sts
done
```

**Note: Attempting to enable pg_tde without properly configuring
shared_preload_libraries will result in an error indicating that pg_tde can
only be loaded at server startup.**

### Secure key management with HashiCorp Vault

While pg_tde supports file-based key storage, it is recommended that production
environments use a Key Management Service such as HashiCorp Vault or OpenBao to
ensure secure key management.

Deploy Vault in your Kubernetes cluster:

```
helm repo add hashicorp https://helm.releases.hashicorp.com
helm install vault hashicorp/vault 
  --disable-openapi-validation 
  --version 0.16.1 
  --namespace vault 
  --set dataStorage.enabled=false 
  --set global.logLevel=trace 
  --set global.platform=kubernetes
```

After deploying Vault, we need to initialize it and obtain the root token. This
process involves creating an initial set of encryption keys and unsealing the
Vault instance:

```
kubectl exec -it vault-0 -- vault operator init 
  -tls-skip-verify 
  -key-shares=1 
  -key-threshold=1 
  -format=json >vault.json

unsealKey=$(jq -r ".unseal_keys_b64[]" <vault.json)
token=$(jq -r ".root_token" <vault.json)

kubectl exec -it vault-0 -- vault operator unseal -tls-skip-verify "$unsealKey"

kubectl exec -it vault-0 -- sh -c 
  "export VAULT_TOKEN=$token && 
   export VAULT_LOG_LEVEL=trace && 
   vault secrets enable --version=1 -tls-skip-verify -path=secret kv"
```

Now, we can configure pg_tde to use Vault as the key provider:

```
SELECT pg_tde_add_key_provider_vault_v2(
  'vault-provider',
  '<rootToken>',
  'http://vault.vault.svc.cluster.local:8200',
  'secret',
  NULL
);

SELECT pg_tde_set_principal_key('tde-principal-key','vault-provider');
```

### Creating encrypted tables

Percona Server for PostgreSQL offers two encryption methods:

1. **tde_heap**: Available exclusively in Percona Server for PostgreSQL 17,
   this method provides comprehensive encryption of tuples, Write Ahead Log
   (WAL), and indexes with optimized performance.
2. **tde_heap_basic**: Compatible with Community PostgreSQL 16 and 17, this
   method encrypts tuples and WAL. Unfortunately this method does not encrypt
   indexes, which is an important limitation to the production use cases.
   Considering the performance limitations of this method encrypting/decrypting
   data whenever it enters/exits shared buffers it is not recommended to use this
   method in production use cases.

When using Percona Server for PostgreSQL images, we can utilize the more comprehensive tde_heap method:

```
CREATE TABLE encrypted_data (
  id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  t text not null
) USING tde_heap;
```

Verify the encryption status of your table:

```
SELECT pg_tde_is_encrypted('encrypted_data');
```

For more detailed information about pg_tde capabilities and configurations,
please refer to the [official Percona documentation](https://percona.github.io/pg_tde/main/index.html).

### Current implementation status and future plans

In the current release of Percona Operator for PostgreSQL, implementing pg_tde
requires manual configuration steps as outlined above. Users need to explicitly
configure shared libraries, set up key providers, and manage the encryption
infrastructure themselves. This approach, while functional, requires careful
attention to detail and a thorough understanding of both pg_tde and Kubernetes
operations.

However, future releases of the Percona Operator will introduce seamless
integration with pg_tde, significantly simplifying the encryption
implementation process. The operator will handle the underlying configuration
automatically, allowing users to focus primarily on their database design. In
these upcoming versions, enabling encryption will be as straightforward as
creating tables with the tde_heap access method while the operator manages all
the necessary infrastructure and configuration behind the scenes.
