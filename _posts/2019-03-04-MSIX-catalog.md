---
layout: post
title: MSIX Catalog
date: "2019-03-04T22:40:32.169Z"
categories: tools windows
---



I've started a desktop application to explore different **modernization** characteristics
of a Windows Desktop application such as Windows 10 api usage, port WPF to .NET Core 3, compare
different deployment methods and use best DevOps practices.

I could create a dummy app to explore these topics, but rather I'd became with the idea
of a small utility to query the catalog of applications installed with MSIX/APPX.

### [https://github.com/ridomin/msix-catalog](https://github.com/ridomin/msix-catalog)

## Explore Windows MSIX management APIs

The MSIX catalog is available in the Windows.Management.PackageManager api. This api allows to load
all the packages installed to the current user, each package can be categorized based on the
kind of signature used, such as Store, Developer, Enterprise or None.

### Windows 10 Contract as NuGet packages

The Windows 10 apis are available with the Windows 10 SDK as contracts in the `winmd` format.

## WPF for .NET Framework and .NET Core 3 

Windows native applications can be written in .NET (WinForms or WPF), C++ (MFC, ATL) and other languages. 
I choosed WPF because this is the team I belong to, but certainly this app could be written to Electron..

WPF is being reivisted as part of the .NET Core 3 release that includes Desktop Workloads, so I'm using this app to learn
different strategies to mantain two versions of the app, for .NET Framework and .NET Core 3.

## Dev Ops for clients apps

In the last years DevOps have gained popularity, specially in server-side workloads, however I've not seen much
content on client applications. This application helps me understand which are the major challenges when building
and deploying applications to client machines. I'm using Azure Dev Ops and AppCenter to manage the lifecycle of the app.


