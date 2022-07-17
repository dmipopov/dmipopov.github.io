---
title: "Hide Kubernetes secrets with helm-secrets, sops and vals"
date: 2022-03-31T22:15:12+03:00
tags: ["Kubernetes", "Helm"]
---


## Methods of secrets delivery

Secrets need to be somehow delivered to Kubernetes clusters. There are two different approaches for this: *pull* and *push*.

### Pull

Secrets are delivered to clusters after deployment. For instance, if we use Helm Charts, we can specify metadata instead of the secret value.

We do not add secret values to the manifests at all, but specify only metadata. Secrets are created based on this metadata and downloaded from external storage -- for example, *Hashicorp Vault*.

### Push

Secrets are delivered to clusters immediately during deployment. We specify secrets in the manifests when creating resources with secrets in the process of deployment.

The easiest way to store secrets safely in repository -- encrypt them in manifests or in Helm Charts. And we decrypt them only when we apply our manifests. Or after apply, if we use an intermediate k8s resource.

The example below describes *push* method with Helm plugin *helm-secrets*, *sops* and *vals*. This combination of tools allow us to safely store secrets in git repository and easily deploy Helm Charts with decrypted secrets without *Hashicorp Vault* installation and maintenance.

## Sops

[Sops](https://github.com/mozilla/sops) allows us to encrypt YAML/JSON files with AWS KMS, GCP KMS, Azure Key Vault and PGP.

In this example we will be using PGP key. Let's generate it:

```bash
$ gpg --gen-key
```
Fill in your full name and email and you'll see:
```bash
...
public and secret key created and signed.

pub   ed25519 2022-03-30 [SC] [expires: 2024-03-29]
      770281CBDD9AD22235A1790DC10C6C36DA3C6597
uid                      Dmitry Popov <dmipopov01@gmail.com>
sub   cv25519 2022-03-30 [E] [expires: 2024-03-29]
```

You can list all your keys with `gpg --list-keys`
```bash
...
/home/dima/.gnupg/pubring.kbx
-----------------------------
pub   ed25519 2022-03-30 [SC] [expires: 2024-03-29]
      770281CBDD9AD22235A1790DC10C6C36DA3C6597
uid           [ultimate] Dmitry Popov <dmipopov01@gmail.com>
sub   cv25519 2022-03-30 [E] [expires: 2024-03-29]

pub   ed25519 2022-03-30 [SC] [expires: 2024-03-29]
      907DA4CB86ECFCCE67CF01FF1D5C44EF6E555F1A
uid           [ultimate] Dmitry Popov <dmipopov01@sberbank.ru>
sub   cv25519 2022-03-30 [E] [expires: 2024-03-29]
```
I have 2 keys on my host and we can use different keys for dev- and prod-environment.

Let's create secrets that we want to encrypt:

*secrets.dev.yaml*

```yaml
dev:
  redis_password: "amazing_redis_password"
  app_api_key: "1a2b3c4d"
```

*secrets.prod.yaml*

```yaml
prod:
  redis_password: "shhhhh_redis_password"
  app_api_key: "12312312312121231312313"
```

Now let's create *.sops.yaml* to specify file masks and PGP keys for them:

```yaml
creation_rules:
  - path_regex: \.*dev*\.yaml$
    pgp: 770281CBDD9AD22235A1790DC10C6C36DA3C6597
  - path_regex: \.*prod*\.yaml$
    pgp: 907DA4CB86ECFCCE67CF01FF1D5C44EF6E555F1A
```

Let's encypt our dev secrets:

```bash
$ sops -e secrets.dev.yaml > enc.secrets.dev.yaml
```

```bash
$ cat enc.secrets.dev.yaml

dev:
    redis_password: ENC[AES256_GCM,data:CsIjY1m02nXRbEmwbRuRlOjA3XAH0A==,iv:KIDiAkJHk/iOfaQmQFneIae3bMM21nwHM+3XQueq0/I=,tag:FGSDIO45xV4F+i6etL03sA==,type:str]
    app_api_key: ENC[AES256_GCM,data:JZguc/TXYKM=,iv:so3U6MQekqaz4HKZ9c/+Q8VAFRUbqr8mXwE+kp5gdt0=,tag:D69EWGAKkOxg6Brm6zIJqw==,type:str]
sops:
    kms: []
    gcp_kms: []
    azure_kv: []
    hc_vault: []
    age: []
    lastmodified: "2022-03-30T20:39:21Z"
    mac: ENC[AES256_GCM,data:Yqmx+PagcpPH7VM5pbq+9Z3c5bhhtMb1vkIgoHn9TLHeJylT7vRrXLYCjVzR9jNFaWtPo8voToF9xbBXfAtHBkW2xgAe3tG/ckOFSf0+SH2I3ReCKbdUOn5TTiIFZJhSZSYruCgjPwvHH6Ghd0qA8tVMKKF8eir50h7G8/mGnSs=,iv:lltyB5VyzgHkmlgobqh/Fi7NlqjV9wLICrm9qQLY/84=,tag:Kbyy37K9kN9+8Fpyk7eGzw==,type:str]
    pgp:
        - created_at: "2022-03-30T20:39:00Z"
          enc: |
            -----BEGIN PGP MESSAGE-----

            hF4DFRPRFvydS5gSAQdAXkkCfQJ4+GcAl4r8v/6GaIYXbqK34nsWzGbK9bCwsQUw
            z4ibV8tO2OGjRDxgRlEwCaua33pqO98uhz8UYb2q6UQOGjMWNf7yk6e4ZnNXR5n0
            1GgBCQIQGvPhXe71G8As/VY11fxtAynLrQyVC+M6zveyfA7HU9RuBQBJir6frmY8
            6v0k3wjVWR9YfezCr/dl9QreAy/vYgmgbDI3+D0zxIrw/h55WKGVlgRYSrAetg+f
            Rs5tFC/Ugdhxrg==
            =BG+c
            -----END PGP MESSAGE-----
          fp: 770281CBDD9AD22235A1790DC10C6C36DA3C6597
    unencrypted_suffix: _unencrypted
    version: 3.7.2
```

As we can see sops saved the structure of *secrets.dev.yaml* and now our secrets are encrpyted.

Command ```sops -d enc.secrets.dev.yaml``` will decrypt and print our secrets.

For in-place editing use `sops -i`. You have to carry keys, but it's easy to import them:
```bash
$ gpg --armor --export-secret-key 770281CBDD9AD22235A1790DC10C6C36DA3C6597 > key-prod.gpg

$ gpg --allow-secret-key-import --import key-prod.gpg
```

## Helm-secrets + sops

[Helm-secrets](https://github.com/jkroepke/helm-secrets) -- is Helm plugin that can be used with sops when we want to deploy Helm Chart.

Installation:

```bash
$ helm plugin install https://github.com/jkroepke/helm-secrets --version v3.12.0
```

Some helm-secrets features:
- Print out secrets with `helm secrets view secrets.dev.yaml`
- In-place encryption with `helm secrets enc secrets.dev.yaml`
- Editing with `helm secrets edit secrets.dev.yaml` by `$EDITOR`
- Automatic rendering of secret values while deploying:
```bash
$ helm secrets upgrade --install some-app -f values.yaml -f secrets.dev.yaml
```

The biggest disadvantage of helm-secrets -- you can't use several drivers at the same time.

## Helm-secrets + sops + vals

[Vals](https://github.com/variantdev/vals) -- tool that allows us to refer to different sources of secrets: sops, vault, aws, gcp.

We'll be using vals as driver for helm-secret.

Using vals is pretty easy -- just store link to secret value directly in `values.yaml`:

```yaml
dev:
  secret1: ref+sops://path/to/file#/foo/bar
  secret2: ref+vault://mykv/foo#/bar?address=https://vault1.example.com:8200
  secret3: ref+gcs://BUCKET/KEY/OF/OBJECT[?generation=ID]
```
**The killer feature of vals is that we can place secrets from different sources in 1 file.**

We can simply view our secrets from `values.yaml` by vals:
```bash
$ helm secrets -d vals view values.yaml -
```

Render our secrets file:
```bash
$ helm secrets -d vals -f values.yaml template .
```

And deploy with decrypted secrets:

```bash
$ helm secrets -d vals upgrade --install some-app -f values.yaml
```

The combination of *helm-secrets*, *sops* and *vals* simplifies management of secrets. It gives the ability to store them safely in git, the only thing to do is to manage GPG keys. And in the future it will be easier for you to migrate from sops to secrets storage such as *Hashicorp Vault* or *AWS Secrets Manager*.
