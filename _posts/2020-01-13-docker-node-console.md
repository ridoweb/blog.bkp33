---
layout: post
title: Docker Node Console
date: "2020-01-13T19:52:01.000Z"
categories: tools
---

I've never have had a good time installing `node` on my dev box, Windows or Mac, I never know which version should I install, or what's the proper configuration to support multiple node versions. I tried `nvm` and others but never got satisfied.

Finally, I discovered how easy is to use `docker` to use the version of node that you need without installing node in your main computer.

Because this I found myself using docker for development almost everyday I was looking how can automate this workflow for any repo that requires node, and finally I found a simple -and I think, elegant -solution:

Create the file `dnc.bat` on `%USERPROFILE%\AppData\Local\Microsoft\WindowsApps` with the next contents:
```
docker run -it -v %cd%:/app -w /app node:latest /bin/bash 
```

Now you can just type `dnc` (that means, *Docker Node Console*) on any folder that requires node to start a linux shell with the latest node version. Of course, you can customize which docker image you want to use.

