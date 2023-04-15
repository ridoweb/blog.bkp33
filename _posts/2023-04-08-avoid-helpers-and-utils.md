---
title: Avoid Helpers and Utils
layout: post
date: 2023-04-08
---

### Why I don't like Helpers and Utils ?

> It's a sign of incomplete design.

I blogged about it some years ago !! 

[Helpers and Utils are Code-Smells (Nov 2005)](https://learn.microsoft.com/en-us/archive/blogs/rido/helpers-and-utils-are-codesmells)


... and I still have a very allergic reaction when see a repo using any of these two words: _helpers_ or _utils_, to mane  classes, methods, namespaces, or any other api surface.


### TL;DR;

The most valuable quality of any application is to describe the architecture by having a good design. The design is not a document, is a combination of class names, folder structures, configuration variables, and deployment scripts. All of these artifacts have to work smoothly, by using the same concepts, in terms of _nouns_ and _verbs_, and every single piece of functionality is well defined with clear inputs and outputs establdhed by contracts.

It might seem unsignificant, but every time I have to browse a folder named _Utils_, to find a dozen of _Helpers_ classes, does not provide any clue of what those _helper* functions might do, or how those work together. This unsignificant detail has huge effects in the maintaibility of the solution. Now, every time you browse the folder, you have to open each individual file to get an idea of what it's purpose is.

In bigger code bases, these might become a nightmare, where every other feature can use those helpers. Now they are tight coupled to their consumers, making future refactoring across the entire codebase.

#### How to organize those _helpers_

Hopefully, at some point you will find (aka emergent design) the right terms to describe the business logic as a cohesive componets exposing an easy-to-use API surface. This is not easy, and often you will find yourself in a position where adding static methods seems ok.

Then try to find a good name, usually a noun to name what kind of functions will be used, with time I've found those classes, with just static methods, very usefull because we don't have to deal with any state.

Just don't call those classes, and files as _helpers_, a common example.

```ts
Helpers.LoadCertificate(path :string) : X509Certificate
```

instead rename to:

```ts
CertificateLoader.LoadFromFile(path :string) : X509Certificate
```

Now you have a file, eg `CertificateLoader.js` that exposes a method to load certificates from files. This file is a single unit, that it's easy to mantain evolve, and at some time, when the design emerges, to make it part of a more complete API. 

### Few Assemblies, Many classes

Yoy

You could argue that now my Helper.cs with 12 methods, is spread 5 files with two or three methods, and now I have to decide if I should put all of those Helpers in a single Assembly, so now I'm appying separation of concerns.

This is also wrong. The deployment has a different complete goal that the source code design. Now we need to have as fewer assemblies as possible, start with one, and have a very thoughtfull process before adding the second assmbly, and always always keep an eye to avoid the typical _assembly_explosion_ we've always seen.

So as a general rule:

> as few assemblies as possibe, and as many classes as necessary



