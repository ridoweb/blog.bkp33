---
title: The IoT Pattern. A dotnet implemetation for MQTT
layout: post
date: 2022-09-18
---

If there is one thing about IoT that can be considered as a pattern, is the characteristics that define a IoT Solution:

    **Devices that can be managed remotely**

The term devices, is very broad, and can be reduced to the idea of _small_ applications, running on different hardware (from constrained MCUs, to complete Windows or Linux computers, and everything in between) that are able to communicate with and endpoint, typically a cloud service.

These devices are able, not only to request information - like we do everyday through HTTP in our browsers - but also to establish a bi-directional communication. And this is what defines the _ IoT Pattern _.

# The IoT Pattern

The patterns consist on 3 main communication behaviors, usually referred as D2C, for Device to Cloud communications, or C2D for Cloud to Device.

## Telemetry

Messages sent from the device to the cloud (aka as D2C) , usually with measurements obtained from sensors, such as _temperature_. Since these measuremens change frequently, the data is considered "volatile", and should be stored somewhere for further analysis. These messages can be as simple as numbers, or following a more complex schema.


## Command

The message is intiated from the solution, or service,  received by the device - a form of C2D message- and optionally returning a response. Both, the request and the response, can use different payloads.


## Property

Devices can describe some of their characteriscts, such as the serial number, or hardware detils, but also what can be referred as the _device state_. Like the variables that define the runtime behavior, such as the how often to send telemetry messages. There are two types of device properties:

### Reported Properties

These properties are _reported_ from the device to the cloud, sometimes also referred as _read only_ properties, and can be defined with:

### Desired Properties

Some properties can be set from the solution side, C2D, where the device should receive the desired property change, and accept or reject the value. These properties are referred to as _desired properties_, or _writable propeties_. The acceptance/rejection of a property can be described with _ack_ messages:


# C# Interfaces describing the IoT Pattern

Telemetry, Property and Command can be described with the next C# interfaces:  

## Telemetry Interface

```cs
public interface ITelemetry<T>
{
    Task<MqttClientPublishResult> SendTelemetryAsync(T payload, CancellationToken cancellationToken = default);
}
```

## Command interface

The best way to describe _incoming_ messages is to use C# delegates, or using modern C# constructs as `Func` to define the callbacks for C2D messages:

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
    Func<T, TasK<TResponse>>? OnCmdDelegate { get; set; }
}
```

> Note: The Request must be able to be Deserialized from the incoming payload, and the response must include not only the payload but also the status


## Property Interface

As described earlier, we must differenciate _ReadOnlyProperties_ and _WritableProperties_

### ReadOnly Property Interface

A property represents the device state and must able to send this state to the service using a D2C message:

```cs
public interface IReadOnlyProperty<T>
{
    string PropertyName { get; }
    T PropertyValue { get; set; }
    Task<int> ReportPropertyAsync(CancellationToken cancellationToken = default);
}
```

### Writable Property Interface

A Writable Property extends the `IReadOnlyProperty<T>` interface by adding a callback to receive desired property updates, and also provides a return value repressnted as a `PropertyAck` to indicate if the property was accepted or rejected: 

```cs
public class PropertyAck<T>
{
    public string Name { get; set; }
    public string? Description { get; set; }
    public int Status { get; set; }
    public T Value { get; set; } 
}

public interface IWritableProperty<T> : IReadOnlyProperty<T>
{
    Func<PropertyAck<T>, PropertyAck<T>> OnProperty_Updated { get; set; }
}
```

# MQTT

There are multiple protocols available to implement IoT devices, although there is one that is widely used, and sometimes referred as the _king_ in the IoT landscape: *MQTT*. Compared to the ominpresent _HTTP_, MQTT has two fundamental benefits: bi-directional communication and power efficiency. These two make the protocol ideal to implement **The IoT Pattern**.

## MQTT Brokers

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

Thanks to C# multicast delegates, we can apply a code pattern to isolate the processing of each incoming message in a single class, to follow the Single-Responsibility principle.

This way, we can have different classes to implement Telemetry, Properties and Commands by publishing and subscribing to a well-known MQTT topics.

# Introducing MQTT Topic Bindings

To implement the interfaces described above using the MQTT protocol we can create _topic bindings_, these are classes that will use specific MQTT topics. Let's assume we want to use the next  MQTT topics:

|Pattern|Direction|Topic|
|-------|---------|-----|
|Telemetry| -> | `device/{clientId}/telemetry` |
|Command (request) | <- | `device/{clientId}/command/{commandName}` |
|Command (response) | -> | `device/{clientId}/command/{commandName}/res` |
|ReadOnlyProperty | -> | `device/{clientId}/props/{propertyName}` |
|WritableProperty (request) | <- | `device/{clientId}/props/{propertyName}/set` |
|WritableProperty (response) | -> | `device/{clientId}/props/{propertyName}/ack` |

> Note: Direction means if the message is published or received from a device point of view: `->` means publish and `<-` means subscribed

> Note: The syntax `{clientId}` and `{commandName}` must be replaced with the actual values

These binders will use Dependency Injection to use an existing MQTT connection, represented by the `IMqttClient` interface available in `MQTTnet`.  

```cs
public interface IMqttClient
{
    Task<MqttClientPublishResult> PublishAsync(topic, payload);
    Task<MqttClientSubscribeResult> SubscribehAsync(topic);
}
```

## Telemetry Topic Binding

The Telemetry class implements the `Telemetry<T>` interface publishing messages to the corresponding topic

```cs
public class Telemetry<T> : ITelemetry<T>
{
    private readonly IMqttClient connection;

    public Telemetry(IMqttClient connection, string name)
    {
        this.connection = connection;
    }

    public async Task<MqttClientPublishResult> SendTelemetryAsync(T payload, CancellationToken cancellationToken = default) =>
        await connection.PublishAsync(
            $"device/{connection.Options.ClientId}/telemetry", 
            payload, 
            Protocol.MqttQualityOfServiceLevel.AtMostOnce, 
            false, 
            cancellationToken);
    
}
```

## Command Topic Binding

Commands will expose a delegate callback and will send the command response:

```cs
public class Command<T, TResponse> : ICommand<T, TResponse>
        where T : IBaseCommandRequest<T>, new()
        where TResponse : IBaseCommandResponse<TResponse>
{
    public Func<T, Task<TResponse>>? OnCmdDelegate { get; set; }

    public Command(IMqttClient connection, string commandName)
    {
        _ = connection.SubscribeAsync($"pnp/{connection.Options.ClientId}/commands/{commandName}");
        connection.ApplicationMessageReceivedAsync += async m =>
        {
            var topic = m.ApplicationMessage.Topic;
            if (topic.Equals($"device/{connection.Options.ClientId}/commands/{commandName}"))
            {
                T req = new T().DeserializeBody(Encoding.UTF8.GetString(m.ApplicationMessage.Payload));
                if (OnCmdDelegate != null && req != null)
                {
                    TResponse response = await OnCmdDelegate.Invoke(req);
                    _ = connection.PublishJsonAsync($"device/{connection.Options.ClientId}/commands/{commandName}/resp/", response.ReponsePayload);
                }
            }
        };
    }
}
```

## Property Binders

Property can be updated with the `UpdateProperyBinder` implementation

```cs
public interface IPropertyStoreWriter
{
    Task<int> ReportPropertyAsync(object payload, CancellationToken token = default);
}

public class UpdatePropertyBinder : IPropertyStoreWriter
{
    private readonly IMqttClient connection;
    private readonly string name;
    
    public UpdatePropertyBinder(IMqttClient connection, string propName)
    {
        this.connection = connection;
        name = propName;
    }

    public async Task<int> ReportPropertyAsync(object payload, CancellationToken cancellationToken = default)
    {
        await connection.PublishAsync(
            $"device/{connection.Options.ClientId}/props/{name}", 
            payload,
            Protocol.MqttQualityOfServiceLevel.AtLeastOnce, 
            true, 
            cancellationToken);
    }
}
```

Similarly desires properties are implemented by a desired property binder follwing the same C2D pattern as commands:

```cs
public class DesiredUpdatePropertyBinder<T>
{
    private readonly IMqttClient connection;
    private readonly string name;

    public Func<PropertyAck<T>, PropertyAck<T>>? OnProperty_Updated = null;

    public DesiredUpdatePropertyBinder(IMqttClient c, IPropertyStoreWriter propertyBinder, string propertyName)
    {
        connection = c;
        name = propertyName;
        c.ApplicationMessageReceivedAsync += async m =>
        {
            var topic = m.ApplicationMessage.Topic;
            if (topic.StartsWith($"pnp/{connection.Options.ClientId}/props/{propertyName}/set"))
            {
                JsonNode desiredProperty = JsonNode.Parse(Encoding.UTF8.GetString(m.ApplicationMessage.Payload))!;
                if (desiredProperty != null)
                {
                        var property = new PropertyAck<T>(propertyName, componentName)
                        {
                            Value = desiredProperty.Deserialize<T>()!,
                        };
                        var ack = OnProperty_Updated(property);
                        if (ack != null)
                        {
                            _ = propertyBinder.ReportPropertyAsync(ack);
                        }
                }
            }
            await Task.Yield();
        };
    }
}
```

## Read Only  Property

Using he property binders, the implementation for ReadOnly and Writable ony interfaces can be implemented as

```cs
public class ReadOnlyProperty<T> : IReadOnlyProperty<T>
{
    private readonly IPropertyStoreWriter updateBinder;
    public string PropertyName { get; }
    public T PropertyValue { get; set; }

    public ReadOnlyProperty(IMqttClient connection, string name)
    {
        updateBinder = new UpdatePropertyBinder(connection, name);
        PropertyName = name;
        PropertyValue = default!;
    }

    public async Task<int> ReportPropertyAsync(CancellationToken cancellationToken = default)
    {
        await updateBinder.ReportPropertyAsync(PropertyValue!, cancellationToken);
        return -1;
    }
}
```

## Wrtiable Property

```cs
public class WritableProperty<T> : IWritableProperty<T>
{
    private readonly IMqttClient connection;
    private readonly string propertyName;
    private readonly IPropertyStoreWriter updatePropertyBinder;
    private readonly DesiredUpdatePropertyBinder<T>? desiredBinder;
    
    public string PropertyName => propertyName;
    public PropertyAck<T> PropertyValue { get; set; }


    public WritableProperty(IMqttClient c, string name, string component = "")
    {
        connection = c;
        propertyName = name;
        updatePropertyBinder = new UpdatePropertyBinder(c, name);
        PropertyValue = new PropertyAck<T>(name, componentName);
        desiredBinder = new DesiredUpdatePropertyBinder<T>(c, updatePropertyBinder, name, componentName);
    }

    public async Task<int> ReportPropertyAsync(CancellationToken token = default) => 
        await updatePropertyBinder.ReportPropertyAsync(PropertyValue, token);
 
    public Func<PropertyAck<T>, PropertyAck<T>> OnProperty_Updated
    {
        get => desiredBinder?.OnProperty_Updated!;
        set => desiredBinder!.OnProperty_Updated = value;
    }
}
```

# Sample Usage

A concrete device will expose an interface compose of the unerlying interfaces described:

```cs
public interface IMqttDevice
{
    public IMqttClient Connection { get; }
    
    public IReadOnlyProperty<string> Property_sdkInfo { get; set; }
    public IWritableProperty<int> Property_interval { get; set; }
    public ITelemetry<double> Telemetry_temp{ get; set; }
    public ICommand<Cmd_echo_Request, Cmd_echo_Response> Command_echo { get; set; }
}
```

And this interface can be implementsd in different ways targeting different MQTT implementations.

## Implementing the interface for a generic broker

