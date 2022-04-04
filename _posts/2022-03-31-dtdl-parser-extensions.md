---
layout: post
title: DTDL Parser extensions
date: '2022-03-31T19:52:01.000Z'
categories: tools
published: true
---

The DTDL dotnet parser provides and object model to inspect DTDL elements: Telemetry, Properties, Commands, Components and Relationships.

All of these are represented as `DTEntityInfo` elements that must be used with the appropriate `DT*` types using casting.

To make it easier the navigation through those types, and by using some C# goodness, I've created this DTDL parser C# extensions:

{% gist 3364aa967ffc66ec503511bc6f01dcfd %}

That can be used as

```cs
var model = await new ModelParser().ParseAsync(ReadFile("dtmi/samples/aninterface-1.json"));

Console.WriteLine(model.Id);

foreach (var t in model.Telemetries)
{
    Console.WriteLine($" [T] {t.Name} {t.Schema.Id}");
}

foreach (var p in model.Properties)
{
    Console.WriteLine($" [P] {p.Name} {p.Schema.Id}");
}

foreach (var c in model.Commands)
{
    Console.WriteLine($" [C] {c.Name} {c.Request.Id} {c.Response.Id}");
}
```
