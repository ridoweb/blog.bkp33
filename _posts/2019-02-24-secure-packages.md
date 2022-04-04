---
layout: post
title: Secure NuGet Packages
date: "2019-02-24T12:40:00.149Z"
categories: tools security
---

The [secure-packages-demo](https://github.com/ridomin/secure-packages-demo) repo includes a dummy project with a custom configuration to require that all packages are signed with known certificates. These certificates can be trusted even if they don't chain to a trusted root.