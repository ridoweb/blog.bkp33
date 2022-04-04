---
layout: post
title: My first 3 days with Azure-IoT
date: "2019-05-31"
categories: iot
---

This week I've started in a new team, the **Azure-IoT End-To-End** team. In my first three days I've been exploring the Getting Started experiences using different devices: the MXChip and AzureSphere. I decided to start with the MXChip Az3166 using the *Getting Started* instructions available in https://aka.ms/iot-devkit.

## Connect DevKit AZ3166 to Azure IoT Hub

https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-arduino-iot-devkit-az3166-get-started

The getting started guide explains "How" to perform some tasks such as creating an IoT Hub, connect the device and send telemetry, but it does not explain the "Why". 

> After completing the excersise I was not able to find the telemetry data.

### Content and structure

The quickstart starts preparing the hardware, then it switches to provising the Azure resources like the IoT hub, and the device itself, then preparing the development environemtn and finally creating the project.



#### Prepare the hardware. 
Just explains how to connect the board, and different ports used. However the firmware update is described in the next section. I missed instructions to know which firmware version is installed and to decide if the update is really needed. Connect to the WiFi should be part of the prepare setup. 


#### Setup IoT Hub, register devices

The IoT Hub creation is described with screenshots, while the device registration uses the Azure CLI (aka `az`)
Configuring the WiFi, or updated the firmware should not be part of this section.

> I tried to use PowerShell, but I could not find the cmdlets to register devices on IoT Hub.

#### Preparing the development environment

I got confused for all the tools I installed on my machine, without any clue on why it's needed or how to verify if the installation succeeded.

- Arduino. In the guide introuction I read *Arduino-compatible board* but for a newbie it's not clear what it means, and how it will be used. Once I got Arduino installed, do I need anything else?. What's the value of VSCode (more on that later).

- Arduino Code extension. Why we need it? What's the value of that extension? It was not easy to find since there was no links to the extension itself. https://marketplace.visualstudio.com/items?itemName=vsciot-vscode.vscode-arduino, or the repo https://github.com/Microsoft/vscode-arduino 

- VS Code extension(s). Although the tutorial requires the Azure IoT tools https://aka.ms/azure-iot-tools, and metnions the relationship with the Workbench and IoT Hub Toolkit, it does not explains the full extension dependency chain resulting in getting more extensions that I need for this tutorial, e.g. Azure IoT Edge.

- Step 5. Configure VS Code with Arduino settings has a typo. The VS Code settings are no longer available under the *Open settings json*. 

> Configuring the Arduino path should be automated.

- Step 6. Configuring AZ3166 does not describes any version requirements, and the screenshot needs an update to show the latest version.

> The links in the *Arduino Board Manager - MXChip* (More info, and online help) redirect to the doc I'm reading.

- Installing the ST-Link/V2 drivers. This should be described as a pre-requisite for Windows at the beginning, makes no sense to install the driver at the end if it's needed by Arduino. There is no mention for versioning.

> The direct link to get the drivers (https://aka.ms/stlink-v2-windows) redirects to OneDrive, this is weird and could cause issues with their Terms-of-Use. 

#### Building my first project

I personally found not very usefull to see the "Welcome page" with links to some examples. I was expecting to clone the samples from a known repo. 

> The first tutorial has a different look & feel (from docs), than the rest (just MD from GitHub)

- Provision Azure IoT Hub and device. I'm wondering why I created the Hub at the beginning if I didnt use it until now. And why I got this extra 6 steps to do the same thing.

- Configure and compile device code. First step is to configure the serial port, I would expect this task once the code is ready to be deployed/debugged. 

After configured the Hub Connection String and deploy the application to the MXChip, I was able to see the messages in the serial port, however I did not found any activity on the portal:

```
2019-06-01 00:55:19 INFO:  >>>IoTHubClient_LL_SendEventAsync accepted message for transmission to IoT Hub.
2019-06-01 00:55:19 INFO:  >>>Confirmation[5] received for message tracking id = 5 with result = IOTHUB_CLIENT_CONFIRMATION_OK
2019-06-01 00:55:21 INFO:  >>>IoTHubClient_LL_SendEventAsync accepted message for transmission to IoT Hub.
2019-06-01 00:55:21 INFO:  >>>Confirmation[6] received for message tracking id = 6 with result = IOTHUB_CLIENT_CONFIRMATION_OK
```

#### Test the project

- Configure Connection Sting. Why I need to configure the connection string again? 

> Typo, when showing the connection string, the key is highlighted, but then it reuires the full connection string

> Typo, there is no option: *Start Monitoring D2C Message* instead it's named as *Start monitoring Built-in Event endpoint*.

## Other Issues

There is an issue with VS Code showing false problems for the C code, eg:
- for `delay(2000);` shows the line as a problem with squiggles with error: `expression preceding parentheses of apparent call must have (pointer-to-) function type`.
- when navigating the code I've seen errors like: `argument of type \"__va_list_tag *\" is incompatible with parameter of type \"char *\""`
























