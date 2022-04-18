---
title: MQTT Topic bindings for C#
layout: post
date: 2022-04-09
---

The MQTT protocol, based on _PubSub_ requires to define a set of topics that clients will use to publish and subscribe messages. 

Once the client is subscribed to one or more topics, all incoming messages will be processed in the same callback handler, making the code difficult to mantain, as any new incoming message should be processed in the same code block, using MQTTNet4 as a example.

```cs
await mqttClient.SubsbcribeAsync("my/topic");
await mqttClient.SubsbcribeAsync("my/other/topic");
mqttClient.ApplicationMessageReceivedAsync = async m => {
    swtich (m.ApplicationMessage.Topic)
        case: "my/topic"
            //process message
            break;
        case: "my/other/topic"
            //process message
            break;
        default:
            break;
}
```

# Messaging Patterns

Is a common practice to define topics that must be used together to implement different variations of Request/Response, a common example is a `Command`, where a device susbcribes to a topic to get the request, and must publish to another topic with the reply.

Another case happens when there is a service connected to the broker exposing and API to enable clients to send requests to a topic, and the response will be send to a different topic. 

When implementing these two patterns using a single callback handler, the code becomes difficult to mantain and to apply SOLID princples, since now the global handler is responsible to dispatch all incoming messages.

Let's build the following example on top of IoT Hub primitives:

- Devices can request the DeviceTwin by sending a message to `$iothub/twin/GET/?$rid={rid}` and will get the response in `$iothub/twin/res/200`
- Devices can implement commands by susbcribing to `$iothub/methods/POST/#` and must send the command response to `$iothub/methods/res/{response.Status}/?$rid={rid}`

Instead of mixing the device implementation with the MQTT plumbing logic, we will create clases that will be responsible to implement each of the features and expose an API to allow implementing the logic without understanding the underlying MQTT communication.

We will create a `PubSubClient` that will expose a `GetTwin` method and a `OnCommand` delegate, to enable a clear separation of concerns from the code that the user must implement on top of these two concepts:

```cs
var pubSubClient = new PubSubClient(mqttClient);
var twin = await pubSubClient.GetTwinAsync();
pubSubClient.OnCommand = async cmd => {
    Console.WriteLine(cmd.CommandName);
    Console.WriteLine(cmd.CommandPayload);
}
```