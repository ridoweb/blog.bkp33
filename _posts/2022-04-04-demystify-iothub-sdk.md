---
published: true
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

{% gist 03311a7bfa38c7ef4acfbe274a822b41 %}

IoTHub requires all MQTT connections to be protected with TLS 1.2, and can be configured using the `MQTTNetConnectionOptionsBuilder` with the extension method below:

{% gist 63942b7e7e18f15dd0010d0f565e51cb %}


Once you have your IoT Hub, and your device, grab the device crendtials and replace it in the code below, and once you run it you are connected.

```cs
using MQTTnet;
using MQTTnet.Client;
using MQTTnet.Client.Options;

IMqttClient mqttClient = new MqttFactory().CreateMqttClient();
var connAck = await mqttClient.ConnectAsync(new MqttClientOptionsBuilder()
    .WithAzureIoTHubCredentialsSas(
        hostName: "<your-hub>.azure-devices.net",
        deviceId : "<ourdevice>",
        sasKey: "MDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDAwMDA="
    )
    .Build());
System.Console.WriteLine($"connAck resaon: {connAck.ResultCode} IsConnected: {mqttClient.IsConnected}");

```





