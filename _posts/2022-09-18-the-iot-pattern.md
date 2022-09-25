---
title: The IoT Pattern. A dotnet implemetation for MQTT
layout: post
date: 2022-09-18
---

If there is one thing about IoT that can be considered as a pattern, is the characteristics that define a IoT Solution:

    **Devices that can be managed remotely**

The term devices, is very broad, and can be reduced to the idea of _small_ applications, running on different hardware (from constrained MCUs, to complete Windows or Linux computers, and everything in between) that are able to communicate with and endpoint, typically a cloud service.

These devices are able, not only to request information - like we do everyday through HTTP in our browsers - but also to establish a bi-directional communication. And this is what defines the _ IoT Pattern _.

## The IoT Pattern

The patterns consist on 3 main communication behaviors, usually referred as D2C, for Device to Cloud communications, or C2D for Cloud to Device.

### Telemetry

Messages sent from the device to the cloud (aka as D2C) , usually with measurements obtained from sensors, such as _temperature_. Since these measuremens change frequently, the data is considered "volatile", and should be stored somewhere for further analysis. These messages can be as simple as numbers, or following a more complex schema.

The basic interface to describe telemtry messages can be defined as:

```cs
Result SendTelemetry<T>(T  message);
```

## Commands

The message is intiated from the solution, or service,  received by the device - a form of C2D message- and optionally returning a response. Both, the request and the response, can use different payloads.

```cs
Func<T, TResponse> OnCommandReceivedDelegate;
```

### Properties

Devices can describe some of their characteriscts, such as the serial number, or hardware detils, but also what can be referred as the _device state_. Like the variables that define the runtime behavior, such as the how often to send telemetry messages. There are two types of device properties:

### Reported Properties

These properties are _reported_ from the device to the cloud, sometimes also referred as _read only_ properties, and can be defined with:

```cs
Result ReportProperty<T>(T property>
```

### Desired Properties

Some properties can be set from the solution side, C2D, where the device should receive the desired property change, and accept or reject the value. These properties are referred to as _desired properties_, or _writable propeties_. The acceptance/rejection of a property can be described with _ack_ messages:

```cs
struct Ack<T> {
    int Status;
    string Description
    T Value
}

Func<T, Ack<T>> OnReceivePropertyDelegate;
```

## MQTT

There are multiple protocols available to implement IoT devices, although there is one that is widely used, and sometimes referred as the _king_ in the IoT landscape: *MQTT*. Compared to the ominpresent _HTTP_, MQTT has two fundamental benefits: bi-directional communication and power efficiency. These two make the protocol ideal to implement **The IoT Pattern**.

### MQTT Brokers

MQTT is not only a protocol, is also a Pub/Sub messaging system implemented by MQTT Brokers. The most famous is Mosquitto, offered as open source from the Eclipse foundation, but there are many others with different licensing options, from HiveMQ, to VerneMQ, nanoMQ, or HMQ. 

### MQTT Clients

To interact with the broker we need to: 

1. Establish a connection, using different channels such as TCP, or TCP+TLS, and usually providing authentication credentials such as Username/Password or Client Certificates
2. Subscribe to topics, using different wildcard patterns such as `+` or `#`
3. Publish messages to topics

The most famous client is the `mosquitto-client` available in almost every Windows/Linux/Mac OS in the form `mosquitto_pub` and `mosquitto_sub` CLI commands, and there are multiple other options.

To write device application we need libraries implementing the protocol, the most famous one is `Paho` from the Eclipse foundation, but there are manyu others for practically every programming platform.

No matter with language you use, the pseudo-code to connect, publish and subscribe will look similar to:

```c
mqtt.connect()
mqtt.subscribe('topicA')
mqtt.OnMessageReceived = (topic, msg) => {
    if (topic == 'topicA') {
        // process msg  
    }
}
mqtt.publish('topicA', 'sampleMessage')
```

This pattern, although very powerfull, makes the client code difficult to write, since the client must process all the incoming messages in a single location.

# A C# implementation for MQTT

Thanks to C# multicast delegates, we can apply a code pattern to isolate the processing of each incoming message in a single class, to follow the Single-Responsibility principle.

This way, we can have different classes to implement Telemetry, Properties and Commands by publishing and subscribing to a well-known MQTT topics.

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

### Command Binding

The best way to describe _incoming_ messages is to use C# delegates, or the most powerfull typed events with `Func`:

```cs
public interface IBaseCommandRequest<T>
{
    T DeserializeBody(string payload);
}

public interface IBaseCommandResponse<T>
{
    public int Status { get; set; }
    public T ReponsePayload { get; set; }
}

public interface ICommand<T, TResponse>
        where T : IBaseCommandRequest<T>, new()
        where TResponse : IBaseCommandResponse<TResponse>
{
    Func<T, TResponse>? OnCmdDelegate { get; set; }
}
```

> Note: The Request must be able to be Deserialized from the incoming payload, and the response must include not only the payload but also the status


### Property Bindings

As described earlier, we must differenciate _ReadOnlyProperties_ and _WritableProperties_

#### ReadOnly Property Bindings

Properties represent the state of the device, and when using with a Device State service, such as IoT Hub device twins, or AWS IoT Core device shadows, can include a verin.


```cs
public interface IReadOnlyProperty<T>
{
    string PropertyName { get; }
    T PropertyValue { get; set; }
    Task<int> ReportPropertyAsync(CancellationToken cancellationToken = default);
}
```

#### Writable Property binding

```cs
public interface IWritableProperty<T>
{
    string PropertyName { get; }
    PropertyAck<T> PropertyValue { get; set; }

    Func<PropertyAck<T>, PropertyAck<T>> OnProperty_Updated { get; set; }

    Task InitPropertyAsync(string twin, T defaultValue, CancellationToken cancellationToken = default);
    Task<int> ReportPropertyAsync(CancellationToken token = default);
}
```
