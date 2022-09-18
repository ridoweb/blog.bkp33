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

I will refer to those as *T* for telemetry *P* for Properties and *C* for commands, or *TPC*

## MQTT

There are multiple protocols available to implement IoT devices, although there is one that is widely used, and sometimes referred as the _king_ in the IoT landscape: *MQTT*. Compared to the ominpresent _HTTP_, MQTT has two fundamental benefits: bi-directional communication and power efficiency. These two make the protocol ideal to implement TPC.

### MQTT Brokers

MQTT is not only a protocol, is also a Pub/Sub messaging system implemented by MQTT Brokers. The most famous is Mosquitto, offered as open source from the Eclipse foundation, but there are many others with different licensing options, from HiveMQ, to VerneMQ, nanoMQ, or HMQ. 

# A C# implementation for MQTT

The nature of Pub/Sub make the TPC communication pattern somehow complex to implement, not so much for Telemetry messages where a single _Pub_ command is enough, but defintely requiring the ceremony of subscribing to a topic and implement the callback whenever a new message arrives. Tipically requiring to _dispatch_ the messages received for the subscribed topics. A pseudo-code example will look like:

```
mqtt.connect
mqtt.publish topicC sampleMessage
mqtt.OnMessageReceived msg
    if msg.topic == topicA
        // process msg  
    if msg.topic == topicB
        //process msg
mqtt.subscribe topicA
mqtt.subscribe topicB
```

This pattern, although very powerfull, makes the client code coupled in a single point to all message types, since the client must process all the incoming messages in a single location.

Thanks to C# multicast delegates, we can apply a code pattern to isolate the processing of each incoming message in a single class, to follow the Single-Responsibility principle.

This way, we can have different classes to implement the TPC by publishing and subscribing to the MQTT topics used to implement TPC.

## Introducing MQTT Topic Bindings

A MQTT topic binding is a pattern that allows to bind C# methods and callbacks (aka delegates) to specific MQTT topics.

### Telemetry Binding

As this Binding only requires to publish to a given topic, it's the most straight-forward and can be summarized with this interface definition:

```cs
public interface ITelemetry<T>
{
    Task<MqttClientPublishResult> SendTelemetryAsync(T payload, CancellationToken cancellationToken = default);
}
```

### Property Bindings

As described earlier, we must differenciate _ReadOnlyProperties_ and _WritableProperties_

#### ReadOnly Property Bindings

Properties represent the state of the device, and when using with a Device State service, such as IoT Hub device twins, or AWS IoT Core device shadows, can include a verin.


```cs
public interface IReadOnlyProperty<T>
{
    string PropertyName { get; }
    T PropertyValue { get; set; }
    int Version { get; set; }
    Task<int> ReportPropertyAsync(CancellationToken cancellationToken = default);
}
```
This case is the same as telemetry to send messages to a known topic:

