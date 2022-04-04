---
layout: post
title: The Cert Central Protocol
date: "2019-03-14"
categories: security
---

One of the key parts of Cert Central is the push command. The purpose of this command is to enable developers to publish their public keys associated to their account.

Anyone could submit any public key, however CertCentral extracts the public key after verifying a digital signature from a `SignedCMS` object.

## How Cert Central solves this problem

The `ccc push` command list all the certificates in the certificate store `CurrentUser\Personal` that satisfy the 
Code Sigining EKU requirements. Selecting a certificate will start a dialog with the server:

0. Login from the CLI with the API Key obtained after register
1. The client asks the server for a random number 
2. The server stores the random number and returns it to the client
3. The client signs the random number and sends a SignedCMS object
4. The server validates the signature and the stored random number
5. If succeed the server stores the public certificate included in the SignedCMS object

Any user who *trust* in the *CertCentral registry* can search and download  certificates based in a url like:

[https://certcentral.x509.online/api/cert/getusercert?username=ridomin&thumbprint=728511CC02E6A80B45ABC0CC862FEF1BFD9617D7](https://certcentral.x509.online/api/cert/getusercert?username=ridomin&thumbprint=728511CC02E6A80B45ABC0CC862FEF1BFD9617D7)

>Note how the `username` parameter to make the link between the cert and the github account explicit

To add the certificate to the `LocalMachine\TrustedPeople` with a single command execute

```bash
ccc trust -u ridomin -t 728511CC02E6A80B45ABC0CC862FEF1BFD9617D7
```

## Summary

CertCentral only stores public keys when there is a valid signature to guarantee private key ownership. 