---
title: "(Linux) Extract Public Key & Fingerprint from Private"
date: 2018-03-15T00:28:21+04:00
draft: false
toc: false
images:
tags:
  - linux
  - pub
  - fingerprint
---
# Extract Public Key and Fingerprint from Private Key

When managing SSH keys, you may need to extract the public key or obtain the fingerprint of a private key. Below are the commands to accomplish these tasks.

## Get the Public Key

To extract the public key from a private key file, use the following command:

```sh
ssh-keygen -f keyname -y
```
Replace `keyname` with the path to your private key file.

## Get the Fingerprint

### SHA-1 Fingerprint
Convert the private key to DER format and compute the SHA-1 fingerprint:

```sh
openssl pkcs8 -in path_to_private_key -inform PEM -outform DER -topk8 -nocrypt | openssl sha1 -c
```

### MD5 Fingerprint
Extract the public key in DER format and compute the MD5 fingerprint:

```sh
openssl rsa -in path_to_private_key -pubout -outform DER | openssl md5 -c
```

## Summary

- Public Key: `ssh-keygen -f keyname -y`
- SHA-1 Fingerprint: `openssl pkcs8 -in path_to_private_key -inform PEM -outform DER -topk8 -nocrypt | openssl sha1 -c`
- MD5 Fingerprint: `openssl rsa -in path_to_private_key -pubout -outform DER | openssl md5 -c`

These commands help you extract the public key and obtain the fingerprints for verification purposes.