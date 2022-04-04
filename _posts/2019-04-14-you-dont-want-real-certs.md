---
title: You don't want real certificates
date: "2019-04-14T01:28:37"
layout: post
categories: security
---

> Note: This article describes ideas to work code signing certificates. Not applicable for SSL/TLS communications.

A `real cert` is an X509 certificate issued by a Trusted CA. 

There are some details to discuss from the previous statment:

- Trusted CA. These are Certificates authorities that are trusted by default in most operating systems.
- Real Cert. Is a certificate issued by a Trusted CA

Having one of these certificates is wonderful, you can sign binary files such as  executables (.exe), powershell scripts (.ps1), installers (.msi), and the modern packages formats (.msix, .nupkg). These certs are really powerfull, since everything you sign with them is trusted **by default** in millions of machines all over the world. This power gives you a great responsability: you must keep your private keys safe.

Keeping a digital secret safe is a fascinating task. As long you want to keep a digital record of your secret, there is a risk of a leak. Let's talk about the consequences of a leaked private key associated to my legal entity. Yes, rememenber that code signing certificates are issued to individuals or organizations that have legal identities.

Now that I have a precious secret, a key to open millions of computers, I'm becoming a target from the *bad gauys*, they will try to find me to steal that private key. And if they succedd I'm screwed. The FBI will knock my door asking for responsabilities.. "Did you know the latest virus was signed with your key?".

## Sign with private certificates

This is how GPG works. Each author is responsible to publish their public keys they used to sign one or more pieces of software. Big companies like Microsoft, or NodeJS sign their Linux packages with GPG keys that must be explitily trusted on each system.

I believe X509 keys from private cetificates, aka self-signed, can be used to protect content without requiring Public CAs.