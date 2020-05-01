---
title: "Renew SSL Certs on Azure Application Gateway with Gitlab CI"
date: 2020-05-01T23:40:24+03:00
draft: false
---

Renewing SSL certificates on Azure Application Gateway is a regular toil for
me. Whenever I research how to automate it, it felt like everyone uses some
Azure tools that doesn't fit to my liking. I don't want to copy a year old
Powershell script and paste it to an Azure automation account (Oh, I can't even
use an Azure Automation account because I don't have the permission to register
applications in my client's Active Directory). I don't want to provision a
virtual server just to renew a certificate and no, I don't want to use Azure
Devops pipelines. I already have a CI tool that I like and even built some
tools for it!

Today, I sat and didn't get up until work is done! I'll explain my process so
maybe someone luckier than me could find and use it.

## The Problem

We use Let's Encrypt for all our SSL certificate needs (assuming the client
doesn't need PV or EV) and monitor certificates for all of our services. The
services that run on on-premise data centers usually renew their certificates
with good old cron but the services that runs on Azure doesn't have a virtual
machine that I can ssh into. They're all run on several Azure App Service plans
in Docker containers and behind Azure Traffic Manager and Azure Application
Gateway. So whenever our monitoring system alarms me one or many of the
certificates will expire soon I have to get a new certificate and upload it to
Azure manually.

Manual process involves many steps:

1. Create a new container using certbot image from Dockerhub.
2. Attempt to get a new certificate using ‘certbot certonly --manual'
3. Copy and paste validation code to a local file
4. Upload file to a public container on Azure Blob Storage to let Let's Encrypt servers validate my request
5. Get the certificate and export it to pfx file using `openssl`
7. Change old pfx file with new one in our terraform project
8. Run `terraform apply`

It's disgusting and also very error prone. What if I miss some unapplied change
in the terraform plan (have you ever seen the diff of an Application Gateway
change?). It's also undocumented because whenever I try to document it, I
always find myself in how-to-automate-this-shit rabbit hole.

## The Solution

I solved the problem using Gitlab CI pipelines. I wasn't aware of certbot's
`--manual-auth-hook` option before. Once I saw it, rest came easy.

Here is the `.gitlab-ci.yml`:

```yaml
stages:
  - letsencrypt

renew:
  stage: letsencrypt
  image:
    name: certbot/certbot:latest
    entrypoint: [""]
  script:
    - apk update
    - apk add gcc make python3-dev musl-dev openssl-dev
    - python -m venv venv
    - source venv/bin/activate
    - pip install --upgrade pip setuptools wheel
    - pip install azure-cli
    - az login -u $AZ_ACCOUNT_EMAIL -p $AZ_ACCOUNT_PASSWORD
    - az account set --subscription $AZ_SUBSCRIPTION_ID
    - certbot certonly --manual --preferred-challenges=http --manual-auth-hook letsencrypt/blob_acme_challenge.sh -d $DOMAIN -m $CERTBOT_CONTACT_EMAIL --agree-tos --non-interactive --manual-public-ip-logging-ok
    - openssl pkcs12 -export -out $DOMAIN.pfx -inkey /etc/letsencrypt/live/$DOMAIN/privkey.pem -in /etc/letsencrypt/live/$DOMAIN/cert.pem -certfile /etc/letsencrypt/live/$DOMAIN/chain.pem -password env:PFX_PASSWORD
    - az network application-gateway ssl-cert update --resource-group $AZ_RESOURCE_GROUP_NAME --gateway-name $AZ_APP_GATEWAY_NAME --name $DOMAIN --cert-file $DOMAIN.pfx --cert-password $PFX_PASSWORD
  only:
    - web
    - api
```

The official certbot image uses Alpine as its base. So we need a bunch of
packages to install Azure CLI. Then we get the new certificate.
`--manual-auth-hook` do its job by passing validation string and token to our
custom script.

The script is pretty straight forward:

```bash
#!/bin/sh

ACME_CHALLENGE_DIR=.well-known/acme-challenge
ACME_CHALLENGE_PATH="$ACME_CHALLENGE_DIR"/"$CERTBOT_TOKEN"

mkdir -p "$ACME_CHALLENGE_DIR"
echo "$CERTBOT_VALIDATION" > "$ACME_CHALLENGE_PATH"

az storage blob upload \
    --connection-string "$ACME_CHALLENGE_BLOB_CONNECTION_STRING" \
    --container-name "$ACME_CHALLENGE_BLOB_CONTAINER_NAME" \
    --file "$ACME_CHALLENGE_PATH" \
    --name "$ACME_CHALLENGE_PATH"
```

It just uploads the file to our public container. We already have URL maps in
Gateway that points to this container for all listeners. Once certbot gives us
the new certicate, `openssl` exports it to pfx file, and we change the old
certificate with new one.

That's it. Now I (and everyone) can just create a new pipeline using the Gitlab
UI or my tool `gitlabci`.

Naturally, I prefer `gitlabci`:

```
$ gitlabci pipeline create group/project master -e DOMAIN=api.example.com
````

Now, that's a solution that fits to my liking!

