---
title: The IoT Pattern
layout: post
date: 2022-09-18
---

If there is one thing about IoT that can be considered as a pattern, is the characteristics that define a IoT Solution:

_Devices that can be managed remotely_

The term devices, is very broad, and can be reduced to the idea of _small_ applications that are able to communicate with endpoints using the next communication patterns:

Then, there will be service applications, think of web sites, or mobile apps, that can interact with those devices using the following communication patterns:

- *Telemetry*. Messages sent from the device, usually with measurements obtained from sensors, such as _temperature_
- *Properties*. Each device might have some _state_ that can be reported from the device, and sometimes configured Nfrom the service. We can distinguish these two types of properties as _reported_ and _desired_, or sometimes as _ReadOnly_ (when they can only be updated from the device) or _Writable_ (when the property can be set from the service)
- *Commands*. Actions that can be invoked from the service and will be executed from the device.

I will refer to those as :T: for telemetry :P: for Properties and :C: for commands.