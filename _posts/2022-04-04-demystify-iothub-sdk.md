---
date: 2022-04-04
layout: post
title: Demystifying Azure IoT Hub Device SDKs
categories: mqtt
---

This post covers how to access main IoTHub features using the MQTT protocol without using the official Device SDKs, instead it uses a generic MQTT library for dotnet capable of basic MQTT primitives: `Connect`, `Publish` and `Subscribe` to access the Telemetry, Commands and DeviceTwin features.

## Choosing an MQTT Library

We are going to use [MQTTNet](https://github.com/dotnet/mqttnet) as the foundation for this article, although the same concepts can be applied with any other MQTT client.

## Establishing the connection

Azure IoT Hub offers two different authentication method for devices: Shared Security Access (SAS) Tokens and X509 Certificates. Let's start with SAS Tokens.

The MQTT protocol allow clients to connect by specifying a `UserName` and `Password`, along with the `ClientId`. IoTHub requires a specific format for the UserName based on the HubName, DeviceId and the SasToken, and the Password is computed with an HMAC signature adding an expiry string.

The next gist shows how to obtain the UserName and Password from the device connection string based on SAS tokens:

{% gist 8572122aa346e6bedd2ae0de0b95fcd0 SasAuth.cs %}

IoTHub requires all MQTT connections to be protected with TLS 1.2, and can be configured using the `MQTTNetConnectionOptionsBuilder` with the extension method below:

{% gist 8572122aa346e6bedd2ae0de0b95fcd0 MqttNetExtensions.cs %}


Once you have your IoT Hub, and your device , grab the device credentials from the portal or CLI, and replace it in the code below, and once you run it you are connected. See [this instructions](https://docs.microsoft.com/en-us/azure/iot-develop/quickstart-send-telemetry-iot-hub?pivots=programming-language-ansi-c#register-a-device) to create your IoT Hub and register a device.

```cs
using MQTTnet;
using MQTTnet.Client;
using MQTTnet.Client.Options;

IMqttClient mqttClient = new MqttFactory().CreateMqttClient();
var connAck = await mqttClient.ConnectAsync(new MqttClientOptionsBuilder()
    .WithAzureIoTHubCredentialsSas(
        hostName: "<your-hub>.azure-devices.net",
        deviceId : "<your-device>",
        sasKey: "MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDA="
    )
    .Build());
System.Console.WriteLine(Azs$"connAck resaon: {connAck.ResultCode} IsConnected: {mqttClient.IsConnected}");

```

## Sending Telemetry

The first feature to explore is device telemetry, telemetry messages can be routed to other Azure services such as Storage, EventHub or EventGrid. 

To send a telemetry message we just need to publish a message to a well known MQTT topic:

```cs
var pubAck = await mqttClient.PublishAsync(
        $"devices/{mqttClient.Options.ClientId}/messages/events/",
        JsonSerializer.Serialize(new { Environment.WorkingSet }) );

Console.WriteLine($"Telemetry sent with pubAck: {pubAck.ReasonCode}" );
```

To verify the telemetry has been received by hub use the Azure CLI 

`az iot hub monitor-events -n <your-hub>`

## Implement Commands

Devices connected to IoT Hub can receive command invocations by following the next pattern.

- Subscribe to the topic `$iothub/methods/POST/#`
- Process incoming messages and parse the `commandName` `requestIdentifier` and `commandPayload`
- Publish the commadn response to topic `$iothub/methods/res/{response.Status}/?$rid={rid}`

The following code implements this pattern

```cs
await mqttClient.SubscribeAsync("$iothub/methods/POST/#");
mqttClient.UseApplicationMessageReceivedHandler(async m =>
{
    var topic = m.ApplicationMessage.Topic;
    if (topic.StartsWith("$iothub/methods/POST/"))
    {
        var segments = topic.Split('/');
        var qs = HttpUtility.ParseQueryString(segments[^1]);
        _ = int.TryParse(qs["$rid"], out int rid);
        var cmdName = segments[3];
        var cmdPayload = Encoding.UTF8.GetString(m.ApplicationMessage.Payload);

        Console.WriteLine($"Executing command {cmdName} with rid {rid} and payload {cmdPayload}");
        await mqttClient.PublishAsync($"$iothub/methods/res/200/?$rid={rid}", cmdPayload);
    }
});
```

To invoke the command, and see the response from the CLI:

`az iot hub invoke-device-method -n yourhub -d yourdevice --method-name myCommand --method-paylaod "Hello"`

## Properties (aka Device Twins)

Device Twins allows to manage the device state by using the `reported` and `desired` properties pattern. 

### Read DeviceTwin

To read the device twin devices can send a request by  publish a message to `$iothub/twin/GET/?$rid={rid}` and will get the response subscribing to `$iothub/twin/res/200`.

The code below implements this pattern using the same `mqttClient`:

```cs
await mqttClient.SubscribeAsync("$iothub/twin/res/#");
mqttClient.UseApplicationMessageReceivedHandler(m => {
    var topic = m.ApplicationMessage.Topic;
    if (topic.StartsWith("$iothub/twin/res/200"))
    {
        var segments = topic.Split('/');
        var qs = HttpUtility.ParseQueryString(segments[^1]);
        _ = int.TryParse(qs["$rid"], out int rid);
        var twin = Encoding.UTF8.GetString(m.ApplicationMessage.Payload);
        Console.WriteLine(twin);
    }
});
await mqttClient.PublishAsync($"$iothub/twin/GET/?$rid=1");
```

### Update Device Twin

Device properties can be updated from the device and the values will be stored in the `reported` section of the twin. To make the update devices must send a message to the topic `$iothub/twin/PATCH/properties/reported/?$rid=`, as a result of this request devices can subscribe to the topic `$iothub/twin/res/204` to get the updated reported version. Here is the flow:

- Device subscribes to `$iothub/twin/res/204`
- Device publish a message to `$iothub/twin/PATCH/properties/reported/?$rid=`
- Device process the incoming message and extracts the updated version from the response topic

The code below implements this flow:

```cs
await mqttClient.SubscribeAsync("$iothub/twin/res/#");
mqttClient.UseApplicationMessageReceivedHandler(m =>
{
    var topic = m.ApplicationMessage.Topic;
    if (topic.StartsWith("$iothub/twin/res/204"))
    {
        var segments = topic.Split('/');
        var qs = HttpUtility.ParseQueryString(segments[^1]);
        var twinVersion = Convert.ToInt32(qs["$version"]);
        System.Console.WriteLine(twinVersion);
    }
});
await mqttClient.PublishAsync(
    "$iothub/twin/PATCH/properties/reported/?$rid=2",
    JsonSerializer.Serialize(new { myProperty = "myValue" }));
```

To check the twin value with the CLI: `az iot hub device-twin show -n yourhub -d yourdevice`

### Handle desired property updates

Every time the `desired` section of the twin gets updated, IoT Hub makes the update available to connected devices who are subscribed to the topic `$iothub/twin/PATCH/properties/desired/#`, this flow is similar to the one implemented above for commands:

- Device subscribe to `$iothub/twin/PATCH/properties/desired/#`
- When new message is published to this topic the device reads the payload
- Optionally the device reports back a reported property to indicate the property was accepted or rejected

```cs
await mqttClient.SubscribeAsync("$iothub/twin/PATCH/properties/desired/#");
mqttClient.UseApplicationMessageReceivedHandler(async m =>
{
    var topic = m.ApplicationMessage.Topic;
    if (topic.StartsWith("$iothub/twin/PATCH/properties/desired"))
    {
        string msg = Encoding.UTF8.GetString(m.ApplicationMessage.Payload);
        System.Console.WriteLine(msg);
    }
});
```
To trigger a device update from the CLI: `az iot hub device-twin update -n yourhub -d yourdevice --desired "{'propName':value}"`

## Conlusion

You can access main IoTHub features with any MQTT client by connecting with the approprate crendentials and pub/sub to the  predefined topics to implement device-to-cloud and cloud-to-device patterns.

### Considerations

When using SaS tokens, keep in mind that tere is a default expiry interval - in this sample is 60 minutes, after that period the client will be disconnected. You can use a timer to manually re-connect before the client gets disconnected.

In a future port I will cover how to connect with X509 Certificates that dont have any expiration limt.

The `UseApplicationMessageReceivedHandler` method, when called multiple times, will remove the previous handlers, in a full exmaple you must parse all incoming messages from the same handler, or... (spoiler alert) use a multicast delegate as will show in a future post.

Enjoy Azure IoT Hub without an official SDK.

The complete code for this sample can be found in this [gist](https://gist.github.com/ridomin/8572122aa346e6bedd2ae0de0b95fcd0) 
