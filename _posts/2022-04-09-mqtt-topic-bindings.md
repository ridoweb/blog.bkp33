---
title: MQTT Topic bindings for C#
layout: post
date: 2022-04-09
---

In [my previous post](/mqtt/2022/04/04/demystify-iothub-sdk/) I explained how to access Azure IoT Hub features using any MQTT Client by processing incoming messages using a single `UseApplicationMessageReceivedHandler` to process all incoming messages. This handler implies that I need to implement all features in a single callback method, what is not very nice in terms of single reposibility principle.

In this post I'm going to explore how a [C# multicast delegate](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/delegates/how-to-combine-delegates-multicast-delegates) can help to isolate different communication patterns - for properties and commands - in different classes without clashing the global handler.

## BYO MQTT

Before going deep with the final solution let's discuss another interesting pattern: _BYO MQTT_, basically what it means is to create an anticorruption layer - also known as the adapter pattern - to isolate the required APIs from any specific MQTT Client implementation.

Any MQTT Library should expose an API to Pub/Sub to topics and receive the incoming messages, in its simpliest form it might look like:

```cs
public class PubSubMessage
{
    public string Topic { get; set; } = "";
    public string Payload { get; set; } = "";
}

public interface IPubSubClient
{
    event Func<PubSubMessage, Task> OnMessage;
    Task<int> PublishAsync(PubSubMessage message, int qos = 1, CancellationToken token = default);
    Task<int> SubscribeAsync(string topic, CancellationToken token = default);
}
```
Based on this interface we can create adapters using different MQTT libraries such as MQTTNet 3:

```cs
namespace Rido.Mqtt.MqttNet3Adapter
{
    public class MqttNetClient : IPubSubClient
    {
        public event Func<MqttMessage, Task> OnMessage;

        private readonly MqttClient client;
        public MqttNetClient(MqttClient client)
        {
            this.client = client;
            this.client.UseApplicationMessageReceivedHandler(async m =>
            {
                await OnMessage?.Invoke(
                     new PubSubMessage()
                     {
                         Topic = m.ApplicationMessage.Topic,
                         Payload = Encoding.UTF8.GetString(m.ApplicationMessage.Payload ?? Array.Empty<byte>())
                     });
            });
        }

        public async Task<int> SubscribeAsync(string topic, CancellationToken token = default)
        {
            var res = await client.SubscribeAsync(
                new MqttClientSubscribeOptionsBuilder()
                    .WithTopicFilter(topic)
                    .Build(), 
                token);
            var errs = res.Items.ToList().Any(x => x.ResultCode > MqttClientSubscribeResultCode.GrantedQoS2);
            if (errs)
            {
                throw new ApplicationException("Error subscribing to " + topic);
            }
            return 0;
        }
    // full implementation omitted for clarity
    }
}
```
Note how the `OnMessage` delegate is invoked inside the callback handler.

## Implement messaging patterns with Single Responsibility Principle

We have identified the next messaging patterns for each IoT Hub feature:

- Fire and Forget for *Telemetry*
- Request/Response for *GetTwin* and *Update Twin*
- Response/Request for *Commands* and *Desired Properties Updates*

I'd like to implement each of these patterns in a `HubClient` class, that will implement the interface:

```cs
 public interface IHubMqttClient
{
    Func<GenericCommandRequest, Task<CommandResponse>> OnCommandReceived { get; set; }
    Func<JsonNode, Task<GenericPropertyAck>> OnPropertyUpdateReceived { get; set; }
    Task<string> GetTwinAsync(CancellationToken cancellationToken = default);
    Task<int> ReportPropertyAsync(object payload, CancellationToken cancellationToken = default);
    Task<int> SendTelemetryAsync(object payload, CancellationToken t = default);
}
```

### Commands

The client must susbcribe to the required topic, parse the incoming message, and expose an event to allow clients to respond to the command. These three steps must be encapsulated in a `IoTHubClient` class to allow clients subscribe to the `OnCommandReceived` without need to know the underlying topic structure. The client, using the adapter, should look like: 

```cs
MqttClient mqtt = new MqttFactory().CreateMqttClient();
var connAck = await mqtt.ConnectAsync( new MqttClientOptionsBuilder()
    .WithAzureIoTHubCredentials(hostname, deviceId, SasToken)
    .Build(), cancellationToken);

var client = new HubMqttClient(mqtt) 

client.OnCommandReceived = async m => {            {
    return await Task.FromResult(new CommandResponse()
    {
        Status = 200,
        ReponsePayload = JsonSerializer.Serialize(new { myResponse = "whatever" })
    });
};
```

The implementation of the `HubMQttClient` exposing the `OnCommandReceived` event is shown below:

```cs
public abstract class BaseCommandResponse
{
    [JsonIgnore]
    public int Status { get; set; }
}
public class CommandResponse : BaseCommandResponse
{
    public string ReponsePayload { get; set; }
}
public class GenericCommandRequest
{
    public string CommandName { get; set; }
    public string CommandPayload { get; set; }
}

public class Command<T, TResponse> : ICommand<T, TResponse> where T : IBaseCommandRequest<T>, new()
        where TResponse : BaseCommandResponse
{
    public Func<T, Task<TResponse>> OnCmdDelegate { get; set; }
   public class GenericCommand
    {
        public Func<GenericCommandRequest, Task<CommandResponse>> OnCmdDelegate { get; set; }
        public GenericCommand(IPubSubClient connection)
        {
            _ = connection.SubscribeAsync("$iothub/methods/POST/#");
            connection.OnMessage += async m =>
            {
                var topic = m.Topic;
                if (topic.StartsWith($"$iothub/methods/POST/"))
                {
                    var segments = topic.Split('/');
                    var cmdName = segments[3];
                    string msg = m.Payload;
                    GenericCommandRequest req = new GenericCommandRequest()
                    {
                        CommandName = cmdName,
                        CommandPayload = msg
                    };
                    if (OnCmdDelegate != null && req != null)
                    {
                        (int rid, _) = TopicParser.ParseTopic(topic);
                        CommandResponse response = await OnCmdDelegate.Invoke(req);
                        _ = connection.PublishAsync($"$iothub/methods/res/{response.Status}/?$rid={rid}", response);
                    }
                }
            };
        }
    }
}
```
To wire up this classes, the `HubMqttClient` is implemented using the `IPubSucClient` injected into the constructor:

```cs
public class HubMqttClient : IHubMqttClient
{

    private readonly GenericCommand command;
    public HubMqttClient(IMqttBaseClient c)
    {
        command = new GenericCommand(c);
    }

    public Func<GenericCommandRequest, Task<CommandResponse>> OnCommandReceived
    {
        get => command.OnCmdDelegate;
        set => command.OnCmdDelegate = value;
    }
    // implementation skipped for clarity
}
```