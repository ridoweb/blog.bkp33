---
title: The Microsoft GPG Key
date: "2019-10-09T20:38:00"
layout: post
categories: security
---

When you try to install Microsoft packages on Linux (some examples are VSCode, SQLServer, DotNet or the Moby engine for IoTEdge) you must trust the microsoft GPG key to your system before installing.

The GPG key is obtained from a known URL:

```
https://packages.microsoft.com/keys/microsoft.asc
```

Usually you install the key with a command like:

```
curl https://packages.microsoft.com/keys/microsoft.asc | gpg --dearmor > microsoft.gpg
sudo cp ./microsoft.gpg /etc/apt/trusted.gpg.d/
```

If you dont install the key, but try to install those packages you will get errors like:

```cmd
Err:5 https://packages.microsoft.com/ubuntu/18.04/multiarch/prod bionic InRelease
  The following signatures couldn't be verified because the public key is not available: NO_PUBKEY EB3E94ADBE1229CF
Reading package lists... Done
W: GPG error: https://packages.microsoft.com/ubuntu/18.04/multiarch/prod bionic InRelease: The following signatures couldn't be verified because the public key is not available: NO_PUBKEY EB3E94ADBE1229CF
E: The repository 'https://packages.microsoft.com/ubuntu/18.04/multiarch/prod bionic InRelease' is not signed.
N: Updating from such a repository can't be done securely, and is therefore disabled by default.
```

If the world (I mean the cloud) runs on Linux, and Linux packages are signed with GPG keys that must be explicitely trusted... why we don't use X509 certificates as GPG keys to sign MSIX/NuGet packages? 
Long live to [X509Online](https://bit.ly/x509online).

