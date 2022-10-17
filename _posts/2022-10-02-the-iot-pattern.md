---
title: The IoT Pattern. A .NET implementation for MQTT
layout: post
date: 2022-10-02
---

If there is one thing about IoT that can be considered as a pattern, is the characteristic that define a IoT Solution:

> _Devices that can be managed remotely_

The term devices, is very broad, and can be reduced to the idea of _small_ applications, running on different hardware (from constrained MCUs, to complete Windows or Linux computers, and everything in between) that are able to communicate with and endpoint, typically a cloud service.

These devices are able, not only to request information - like we do everyday through HTTP in our browsers - but also to establish a bi-directional communication. And this is what defines the _ IoT Pattern _.

In this article we are going to explore a .NET implementation of the IoT Pattern for MQTT, using MQTTnet.

# The IoT Pattern

The pattern consists on 2 main communication behaviors, usually referred as D2C, for Device to Cloud communications, or C2D for Cloud to Device.

To represent messages sent from the device to the cloud, we can define the next interface.

```cs
public interface IDeviceToCloud<T>
{
    Task SendMessageAsync(T payload);
}
```

Messages sent from the cloud to the device usually are followed by another message from the device to acknowledge the message was received. For this case we are going to use a callback with a request and a response using a `Func`.

```cs
public interface ICloudToDevice<T, TResp>
{
    Func<T, Task<TResp>>? OnMessage { get; set; }
}
```

## Telemetry

Telemetry is the most common case, when devices send messages with measurements obtained from sensors, such as _temperature_. Since these measurement change frequently, the data is considered "volatile", and should be stored somewhere for further analysis. These messages can be as simple as numbers, or following a more complex schema.

```cs
public interface ITelemetry<T> : IDeviceToCloud<T> { }
```

## Command

Commands are used to invoke actions on devices, and are one common case of C2D messaging, hence:

```cs
public interface ICommand<T, TResp> : ICloudToDevice<T, TResp> { }
```

## Property

Devices can describe some of their characteristics, such as the serial number, or hardware details, but also what can be referred as the _device state_. Like the variables that define the runtime behavior, suchlike how often to send telemetry messages. There are two types of device properties:

### Reported Properties

These properties are _reported_ from the device to the cloud, sometimes also referred as _read only_ properties, and can be defined with:

```cs
public interface IReadOnlyProperty<T> : IDeviceToCloud<T> { }
```

### Desired Properties

Some properties can be set from the solution side, C2D, where the device should receive the desired property change, and accept or reject the value. These properties are referred to as _desired properties_, or _writable properties_. The acceptance/rejection of a property can be described with _ack_ messages. These _ack_ messages contains additional information about the property, such as the Status, Version or Description, that the device can use to inform to the service if the property was accepted, or not, and related details.

Since the ack messages need to be also reported, the IWritableProperty interface implements D2C and C2D:

```cs
public class Ack<T>
{
    public int? Version { get; set; }
    public string? Description { get; set; }
    public int Status { get; set; }
    public T Value { get; set; } = default!;
}

public interface IWritableProperty<T> : ICloudToDevice<T, Ack<T>>, IDeviceToCloud<Ack<T>>
{
    T? Value { get; set; }
    int? Version { get; set; }
}
```

# MQTT

There are multiple protocols available to implement IoT devices, although there is one that is widely used, and sometimes referred as the _king_ in the IoT landscape: *MQTT*. Compared to the omnipresent _HTTP_, MQTT has two fundamental benefits: bi-directional communication and power efficiency. These two make the protocol ideal to implement **The IoT Pattern**.

## MQTT Brokers

MQTT is not only a protocol, is also a Pub/Sub messaging system implemented by MQTT Brokers. The most famous is Mosquitto, offered as open source from the Eclipse foundation, but there are many others with different licensing options, from HiveMQ, to VerneMQ, nanoMQ, or HMQ. 

### MQTT Clients

To interact with the broker we need to: 

1. Establish a connection, using different channels such as TCP, or TCP+TLS, and usually providing authentication credentials such as Username/Password or Client Certificates
2. Subscribe to topics, using different wildcard patterns such as `+` or `#`
3. Publish messages to topics

The most used client is the `mosquitto-client` available in almost every Windows/Linux/Mac OS in the form `mosquitto_pub` and `mosquitto_sub` CLI commands, and there are multiple other options.

To write device applications we need libraries implementing the protocol, like `Paho` from the Eclipse foundation, but there are many others for practically every programming platform.

No matter which language you use, the pseudo-code to connect, publish and subscribe will look similar to:

```js
mqtt.connect()
mqtt.subscribe('topicA')
mqtt.OnMessageReceived = (topic, msg) => {
    if (topic == 'topicA') {
        // process msg  
    }
}
mqtt.publish('topicA', 'sampleMessage')
```

This pattern, although very powerful, makes the client code difficult to write, since the client must process all the incoming messages in a single location, the callback where we subscribe for incoming.

# Introducing MQTT Topic Bindings

To implement the interfaces described above using the MQTT protocol we can create _topic bindings_, these are classes that will implement the details to work with topics: publishing messages and most interestingly to subscribe and make the incoming messages available as .NET callbacks. The messages are serialized for publishing and deserialized when received. These serialization operations are exposed on generic APIs. 

To integrate with an existing  connection, these binders use Dependency Injection to use an existing connection, represented by the `IMqttClient` interface available in `MQTTnet`.  

```cs
public interface IMqttClient
{
    Task<MqttClientPublishResult> PublishAsync(string topic, byte[] payload);
    Task<MqttClientSubscribeResult> SubscribeAsync(topic);
}
```

## Serializers

Messages are represented as a `byte array`, with multiple options to use different serializers, to customize the serialization format the binders can be initialized with a `IMessageSerializer`, such as JSON, Avro, or Protobuf.

```cs
public interface IMessageSerializer
{
    byte[] ToBytes<T>(T payload, string name = "");
    T? FromBytes<T>(byte[] payload, string name = "");
}
```

### Wrapped Messages

There are cases where we want the message to be _wrapped_ with the name, eg:

```json
{
    "temperature" : 23.32
}
```

Hence, the serializers should know that name to provide the actual value.

## DeviceToCloud Binder

This D2C binder implements the `IDeviceToCloud` interface and is responsible to serialize the incoming message into a byte array and publish to the specified topic, there is a constructor that uses the Json serializer and  can be configured with a different serializer.

```cs
public abstract class DeviceToCloudBinder<T> : IDeviceToCloud<T>
{
    private readonly string _name;
    private readonly IMqttClient _connection;
    private readonly IMessageSerializer _messageSerializer;

    public string TopicPattern = "device/{clientId}/telemetry";
    public bool WrapMessage = false;
    protected bool Retain = false;

    public DeviceToCloudBinder(IMqttClient mqttClient, string name) : this(mqttClient, name, new UTF8JsonSerializer()) { }

    public DeviceToCloudBinder(IMqttClient mqttClient, string name, IMessageSerializer ser)
    {
        _connection = mqttClient;
        _name = name;
        _messageSerializer = ser;
    }

    public async Task SendMessageAsync(T payload, CancellationToken cancellationToken = default)
    {
        string topic = TopicPattern
                            .Replace("{clientId}", _connection.Options.ClientId)
                            .Replace("{name}", _name);
        byte[] payloadBytes;
        if (WrapMessage)
        {
            payloadBytes = _messageSerializer.ToBytes(new Dictionary<string, T> { { _name, payload } });
        }
        else
        {
            payloadBytes = _messageSerializer.ToBytes(payload);
        }
        var pubAck = await _connection.PublishAsync(
            new MqttApplicationMessageBuilder()
                .WithTopic(topic)
                .WithPayload(payloadBytes)
                .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
                .WithRetainFlag(Retain)
                .Build(),
            cancellationToken);

        if (pubAck.ReasonCode != MqttClientPublishReasonCode.Success)
        {
            Trace.TraceWarning($"Message not published: {pubAck.ReasonCode}");
        }
    }
}
```
The abstract class includes two protected members to configure its behavior:

- TopicPattern. To configure the topic to publish to.
- Retained. If the message should published with the retain flag.
- WrapMessages. To create a wrapped version of the message.

## CloudToDevice Binder

The C2D binder will listen to all received messages, and expose a delegate to consumers. When a message is received it will filter based on the configured topic pattern and when matched it will deserialize the incoming message and invoke the delegate. Additionally it will serialize and publish the response. 

> Note: Note that all C2D binders will receive ALL messages, but only those who are sent to the configured topic will be actually processed.

```cs
public abstract class CloudToDeviceBinder<T, TResp> : ICloudToDevice<T, TResp>
{
    private readonly string _name;
    private readonly IMqttClient _connection;

    protected bool UnwrapRequest = false;
    protected bool WrapResponse = false;

    protected bool RetainResponse = false;

    public Func<T, Task<TResp>>? OnMessage { get; set; }

    protected Action<TopicParameters>? PreProcessMessage;

    public CloudToDeviceBinder(IMqttClient connection, string name)
        : this(connection, name, new UTF8JsonSerializer()) { }

    public CloudToDeviceBinder(IMqttClient connection, string name, IMessageSerializer serializer)
    {
        _connection = connection;
        _name = name;

        connection.ApplicationMessageReceivedAsync += async m =>
        {
            var topic = m.ApplicationMessage.Topic;
            if (topic.StartsWith(requestTopicPattern!.Replace("/#", string.Empty)))
            {
                if (OnMessage != null)
                {
                    var tp = TopicParser.ParseTopic(topic);
                    PreProcessMessage?.Invoke(tp);

                    T req = serializer.FromBytes<T>(m.ApplicationMessage.Payload, UnwrapRequest ? _name : string.Empty)!;
                    if (req != null)
                    {
                        TResp resp = await OnMessage.Invoke(req);
                        byte[] responseBytes = serializer.ToBytes(resp, WrapResponse ? _name : string.Empty);

                        string? resTopic = responseTopicPattern?
                            .Replace("{rid}", tp.Rid.ToString())
                            .Replace("{version}", tp.Version.ToString());

                        _ = connection.PublishAsync(
                            new MqttApplicationMessageBuilder()
                                .WithTopic(resTopic)
                                .WithPayload(responseBytes)
                                .WithQualityOfServiceLevel(MqttQualityOfServiceLevel.AtLeastOnce)
                                .WithRetainFlag(RetainResponse)
                                .Build());
                    }
                    else
                    {
                        Trace.TraceWarning($"Cannot parse incoming message name: {_name} payload: {Encoding.UTF8.GetString(m.ApplicationMessage.Payload)}");
                    }
                }
            }
        };
    }

    private string? requestTopicPattern;
    protected string? RequestTopicPattern
    {
        get => requestTopicPattern;
        set
        {
            requestTopicPattern = value?.Replace("{clientId}", _connection.Options.ClientId).Replace("{name}", _name)!;
            _ = _connection.SubscribeWithReplyAsync(requestTopicPattern);
        }
    }

    private string? responseTopicPattern;
    protected string? ResponseTopicPattern
    {
        get => responseTopicPattern;
        set
        {
            responseTopicPattern = value?.Replace("{clientId}", _connection.Options.ClientId).Replace("{name}", _name)!;
        }
    }
}
```

With these two binders we can proceed to implement Telemetry, Properties and Commands.

Let's assume we want to use the next  MQTT topics:

|Pattern|Direction|Topic|
|:------|---------|:----|
|Telemetry| -> | `device/{clientId}/telemetry` |
|Command (request) | <- | `device/{clientId}/command/{commandName}` |
|Command (response) | -> | `device/{clientId}/command/{commandName}/res` |
|ReadOnlyProperty | -> | `device/{clientId}/props/{propertyName}` |
|WritableProperty (request) | <- | `device/{clientId}/props/{propertyName}/set` |
|WritableProperty (response) | -> | `device/{clientId}/props/{propertyName}/ack` |

> Note: Direction means if the message is published or received from a device point of view: `->` means publish and `<-` means subscribed

> Note: The syntax `{clientId}`, `{propertyName}` and `{commandName}` should be replaced with the actual values

# Implementing Telemetry, Properties and Commands

With the D2C and C2D binders we can implement the IoT messaging patterns by inheriting from the abstract classes and configure the topics we want to use for each case.

## Telemetry 

The Telemetry class implements the `Telemetry<T>` interface publishing messages to the corresponding topic

```cs
public class Telemetry<T> : DeviceToCloudBinder<T>, ITelemetry<T>
{
    public Telemetry(IMqttClient mqttClient) : this(mqttClient, string.Empty) { }
    public Telemetry(IMqttClient mqttClient, string name)
        : base(mqttClient, name)
    {
        TopicPattern = "device/{clientId}/telemetry";
        WrapMessage = true;
    }
}
```

## Command 

Commands will expose a delegate callback and will send the command response:

```cs
public class Command<T, TResp> : CloudToDeviceBinder<T, TResp>, ICommand<T, TResp>
{
    public Command(IMqttClient client, string name)
        : base(client, name)
    {
        RequestTopicPattern = "device/{clientId}/commands/{name}";
        ResponseTopicPattern = "device/{clientId}/commands/{name}/resp";
    }
}
```

## ReadOnlyProperty

ReadOnly properties will use the property name in the topic pattern and do not need to wrap the message:

```cs
public class ReadOnlyProperty<T> : DeviceToCloudBinder<T>, IReadOnlyProperty<T>
{
    public ReadOnlyProperty(IMqttClient mqttClient) : this(mqttClient, string.Empty) { }
    public ReadOnlyProperty(IMqttClient mqttClient, string name)
        : base(mqttClient, name)
    {
        TopicPattern = "device/{clientId}/props/{name}";
        WrapMessage = false;
        Retain = true;
    }
}
```

Although some times must be required to send all the properties in a single message,  we could use a generic topic, with wrapped message:

```cs
TopicPattern = "device/{clientId}/props";
WrapMessage = true;
```

## WritableProperty

```cs
public class WritableProperty<T> : CloudToDeviceBinder<T, Ack<T>>, IWritableProperty<T>
{
    readonly IMqttClient _connection;
    readonly string _name;
    public T? Value { get; set; }
    public int? Version { get; set; } = -1;

    public WritableProperty(IMqttClient c, string name)
        : base(c, name)
    {
        _name = name;
        _connection = c;

        RequestTopicPattern = "device/{clientId}/props/{name}/set/#";
        ResponseTopicPattern = "device/{clientId}/props/{name}/ack";
        RetainResponse = true;
        PreProcessMessage = tp =>
        {
            Version = tp.Version;
        };
    }

    public async Task SendMessageAsync(Ack<T> payload, CancellationToken cancellationToken = default)
    {
        var prop = new ReadOnlyProperty<Ack<T>>(_connection, _name)
        {
            TopicPattern = "device/{clientId}/props/{name}/ack",
            WrapMessage = false
        };
        await prop.SendMessageAsync(payload, cancellationToken);
    }
}
```

# Sample Usage

Now that we have implementations for Telemetry, Properties and Commands, we can define a custom client composed of one or more of these elements:

```cs
public class SampleClient
{
    public IReadOnlyProperty<string> Property_sdkInfo { get; set; }
    public IWritableProperty<int> Property_interval { get; set; }
    public ITelemetry<double> Telemetry_temp { get; set; }
    public ICommand<string, string> Command_echo { get; set; }
    
    public SampleClient(IMqttClient c) 
    {
        Property_sdkInfo = new ReadOnlyProperty<string>(c, "sdkInfo");
        Property_interval = new WritableProperty<int>(c, "interval");
        Telemetry_temp = new Telemetry<double>(c, "temp");
        Command_echo = new Command<string, string>(c, "echo");
    }
}
```

This client can be used to implement the device application focusing in the application logic without worrying of the underlying details of the pub/sub messaging or serialization formats.

- Initialize the `SampleClient` with an existing mqtt connection:

```cs
SampleClient client = new SampleClient(mqttClient);
```

- Update a ReadOnlyProperty

```cs
await client.Property_sdkInfo.SendMessageAsync("0.1.0.0", stoppingToken);
```

- Send Telemetry 

```cs
 await client.Telemetry_temp.SendMessageAsync(23.34);
```

- Implement a Command

```cs
client.Command_echo.OnMessage = async req =>
{
    return await Task.FromResult(req + req);
};
```

- Implement a WritableProperty update

```cs
client.Property_interval.OnMessage = async p =>
{
    Ack<int> ack = new()
    {
        Version = client.Property_interval.Version,
        Description = "accepted",
        Status = 200,
        Value = p
    };
    return await Task.FromResult(ack);
};
```

# Summary

Following this pattern you can implement MQTT applications by applying SOLID principles. The abstract binders will handle the communication details, the Telemetry, Command, ReadOnlyProperty and WritableProperty implementations can be configured to match the topic structure and serialization formats, compose these with a client and now your application logic can be implemented without taking care of the protocol details.

This pattern is being used in the [MQTTnet.Extensions.MultiCloud](https://github.com/iotmodels/MQTTnet.Extensions.MultiCloud) project, to create MQTT applications that can work with different cloud providers.