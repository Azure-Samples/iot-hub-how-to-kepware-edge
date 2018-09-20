---
services: iot-hub, iot-edge, Kepware
author: stevebus
reviewer: temandin
---

# Azure IoT Edge - How to connect PTC/Kepware's KepServerEx

These instructions provide the necessary steps to connect PTC/Kepware's KepServerEx to Azure IoT Hub **through** Azure IoT Edge.

## Overview

PTC Kepware's KepServerEx is an industry leader in industrial and manufacturing device connectivity. It has connectivity libraries for a vast array of equipment and is a popular choice for unlocking the data from both new and legacy industrial devices. Kepware provides an IoT Gateway module today that, per their own [instructions](https://www.kepware.com/getattachment/c93c65df-57ea-4e9c-a1e0-2e9a34381d54/mqtt-client-and-microsoft-azure-iot.pdf), can be used to connect to and send data to Azure IoT Hub over the MQTT protocol.  

Many customers have expressed a desire to be able to send the data from KepServerEx through Azure IoT Edge. That allows both for Edge to act as a gatewey through which traffic must flow, thus isolating the KepServerEx further from the Internet, as well as the ability to take advantage of the many IoT Edge capabilities for pre-processing that data on the edge. Examples include the ability to use custom modules or Azure Functions to transform the data, Azure Streaming Analytics to do filtering and aggregation of the data before it goes to the cloud, and Azure Machine Learning for making predictions on the edge, potentially without having to send all of the detailed data to the cloud.

Below are the step-by-step instructions for connecting KepServerEx to IoT Hub through IoT Edge

## Prerequisites

You will need

* an Azure subscription.  If you do not already have one, you can create an Azure account and subscription with free credits [here](https://azure.microsoft.com/en-ca/free)
* an IoT Hub.  If you do not already have one, create one via the instructions [here](https://docs.microsoft.com/en-us/azure/iot-hub/iot-hub-create-using-cli#create-an-iot-hub)
* an Azure IoT Edge device set up as a 'transparent gateway' per the Azure IoT Edge documentation ([linux](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-create-transparent-gateway-linux) or [windows](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-create-transparent-gateway-windows)).  Ensure that IoT Edge is set up correctly for secure MQTT communication by running the following command (if Windows, you may have to install [openssl](https://sourceforge.net/projects/openssl/))
* the Java Runtime Engine (JRE) installed on the Kepware Server.  You can install the JRE from this link:  [https://java.com/en/download/](https://java.com/en/download/)

```bash
openssl s_client -connect [your gateway name]:8883 -CAfile $CERTDIR/certs/azure-iot-test-only.root.ca.cert.pem
```

where [your gateway name] is the name you used in the hostname field of your config.yaml file.  If this does not work, try adding your gateway name and ip address to the hosts file (/etc/hosts on Linux or c:\windows\system32\drivers\etc\hosts on Windows).  In an Azure VM this is the internal, not external ip address.  eg.:

```bash
10.0.0.10 KepGateway
```

If the bottom of the results of that command shows anything other than "Verify return code: 0 (ok)" (see screenshot below), then you'll need to fix that before moving on.

![cert verify](images/cert-verify.png)

* a KepServerEx set up and licensed with Kepware's IoT Gateway plug-in
* Administrative permissions on the KepServerEx (both KepServerEx and the underlying host machine)

## Environment Setup

Before configuring Kepware, we need to take a few environmental setup steps.  

### Certificates

When Kepware connects to IoT Edge, it will do so with MQTT over TLS.  The Edge Hub module will provide it with a server certificate for the TLS connection and the KepServerEx must trust that certificate.

* if you used a production certificate from a company like Baltimore, DigiCert, etc to set up your IoT Edge device, or a corporate certificate based on a Certificate Authority(CA) that your KepServerEx will already trust, you can skip this step
  * if you used the dev/test-only [convenience scripts](https://docs.microsoft.com/en-us/azure/iot-edge/how-to-create-transparent-gateway-linux#certificate-creation) provided by the Azure IoT engineering team, you need to copy the Root CA certificate from your IoT Edge box.  You need to get this certificate to a location from which your KepServerEx can read (either copy local to the KepServerEx or a file share it can reach)  
    * If you used the instructions above, this certificate will be located at $CERTDIR/certs/azure-iot-test-only.root.ca.cert.pem  (where $CERTDIR is the directory in which you created your dev/test certificates).  
  * on the KepServerEx, using the Certificate Manager MMC console, import the IoT Edge Root CA certificate into its "Trust Root Certification Authorities" store.  **Make sure you do this for the 'local computer', and not 'personal'**.

### Name resolution

The KepServerEx must also be able to resolve the fully-qualified domain name (FQDN) of the IoT Edge gateway to its IP address.  This FQDN **must** match the hostname parameter used in your IoT Edge device's config.yaml file.

* if you have DNS infrastructure services available in the environment in which your KepServerEx runs, add a DNS entry for your IoT Edge gateway
* if you do not, or just want to test before you do, you can add a hosts file entry on your KepServerEx for your IoT Edge gateway in the c:\windows\system32\drivers\etc\hosts file

Either way, ensure your KepServerEx can 'ping' your IoT Edge gateway device by this FQDN

### Register Kepware with IoT Hub

Even through Kepware will be physically connecting to IoT Hub via the IoT Edge gateway, it will still authenticate through to IoT Hub.  In order to do so, IoT Hub needs to know about the KepServerEx IoT Gateway agent.  To do this, we create an IoT Device in IoT Hub to represent our KepServerEx agent.

To do this,

* Open the [Azure Portal](http://portal.azure.com) in your browser
* click on the "Cloud Shell" button on the menu bar across the top

![Cloud Shell](images/cloud-shell-menu.png)

* install the azure iot cli extension with the following command

```bash
az extension add --name azure-cli-iot-ext
```

* once installed, create your IoT Device to represent your KepServerEx

 ```bash
 az iot hub device-identity create --device-id [device id] --hub-name [hub name]
 ```

where [device id] is the name you want to use for your IoT Edge (e.g. iotKepware01) and the [hub name] is the short-name of your IoT Hub (e.g. sdbedgeIoTHub)

* Finally, we need to generate a security token (called a "SAS token") for our device.  This will be used as the "password" for the KepServerEx to connect to IoT Hub/IoT Edge.  To create the SAS token, run

```bash
az iot hub generate-sas-token --device-id [device id] --hub-name [hub name] --duraction [duration in seconds]
```

where [device id] is the name of the device you just created, [hub name] is the short-name of your IoT Hub, and [duration in seconds] is the duration, in seconds, that you want your SAS Token to be valid.  For security purposes, Microsoft recommends periodically 'rolling' the SAS token by generating a new one and updating your KepServerEx on a schedule you feel balances security with administrative overhead.  Either way, make note of the duration, as you'll need to update the SAS token before the expiration or your KepServerEx will no longer be able to authenticate to IoT Edge.

A valid SAS token will look similar to the following (with [device id] and [hub name] filled in with your values):

SharedAccessSignature sr=[hub name].azure-devices.net%2Fdevices%2F[device id]&sig=zxiTWD2bGOk0ea66FXwFaNO7cboAQSM3OM3u2UNW5T4%3D&se=1567020425

* Copy your generated SAS Token somewhere from which you can paste it later into your KepServerEx's configuration.  

> NOTE:  in the Kepware instructions for connecting directly to IoT Hub, you may note that there is an option for authenticating to IoT Hub via a self-signed certificate. Today, for IoT Edge, SAS Token authentication is the only option. This may change in the future.

## Kepware Configuration

With the preliminary work done, we are ready to configure our KepServerEx.  

* In the KepServerEx Configuration Tool, expand the tree view on the left
* right-click on IoT Gateway and choose "Add Agent".  This opens a wizard for creating a new IoT Gateway Agent

* on the "New Agent" screen
  * give the agent a meaningful name of your choosing
  * choose "MQTT Client" for the type
  * click Next

* On the "MQTT Client - Broker" screen,
  * enter the following values
    * URL:   ssl://[iot edge device FQDN]:8883
    * Topic:  devices/[device id]/messages/events/
  * note that the url protocol was changed from tcp:// to ssl://.  The [iot edge device fqdn] is the hostname of your IoT Edge device.  The [device id] is the device id you created above to represent your KepServerEx server in IoT Hub.  On the topic, **please note the trailing slash and include it**
  * leave the remaining defaults and click Next

* On the MQTT Client - Security screen
  * enter the following values
    * Client ID:   [device id]
    * Username:  [iothub long name]/[device id]/api-version=2016-11-14
    * Password:  [SAS Token]
  * the [iothub long name] is the full name of your IoTHub, including the .azure-devices.net part.  The [SAS Token] is the SAS Token generated and copied above.  Copy/Paste it in here.  On the Username, for IoT Hub the "api-version" parameter was optional, but it is not for IoT Edge.
  * click "Finish"

Below are a couple of screenshots for comparison. To compare to your entries, right click on the agent you created in the tree view and choose "Properties.."

For comparison with the screenshots, the following values were used in the entry fields above:

* [iot edge device FQDN] -> iotedgegw.local
* [device id] -> iotKepware1
* [iothub long name] -> sdbedgeIoTHub.azure-devices.net

Client Tab:

![client tab](images/kepware-agent-config-broker.png)

Security Tab:

![client tab](images/kepware-agent-config-security.png)

## Choose tag(s) to send

At this point, you've created an IoT Agent, but it is not yet setup to send any data to IoT Edge. If you click on your newly created agent in the left hand tree view, in the right hand screen you'll see "New IoT Item".  Click on that, then navigate to the tag(s) that you want to send to IoT Hub and add them. For example, the following screenshots show setting up the "Channel1.Device1.Tag1" sample tag that comes with the KepServerEx simulator.

![sample tag](images/Kepware-sample-tag.png)

![sample tag2](images/Kepware-sample-tag2.png)

## Validation

Now that we are set up, let's validate everything is working.

* On the KepServerEx Configuration Tool, in the eventlog at the bottom, you should see a successful connection to your IoT Edge device similar to this

![Kepware logs success](images/kepware-logs-successful.png)

* On the IoT Edge device, in the Edge Hub logs you should see a successful connection from the IoT Device that represents your KepServerEx server.

To view the logs, run this command.

```bash
iotedge logs -f edgeHub
```

You should find a section similar to this (but with your [device id])

![edge hub logs](images/edgehub-logs.png)

* Finally, you should be able to see the data flowing into IoT Hub.  To do, so back in the browser, in the Cloud CLI window, run the following command  (filling in your iot hub name)

az iot hub monitor-events -n [iot hub name]

After a few seconds, you should see messages similar to the below start flowing in

![kepware event monitor](images/kepware-event-monitor.png)

## Next Steps

Congratulations!  You've connected your KepServerEx server through IoT Edge to IoT Hub.  Now you can start adding in the additional capabilities of IoT Edge to process the data on the Edge.

This is where your imagination can take over, but a few ideas from other customers

* Custom IoT Edge modules to reformat or otherwise manipulate the messages from devices.  Or potentially to push the data to a local dashboard or 3rd party historian.
* Azure Streaming Analytics on Edge jobs to
  * Filter out data you may not want to send to the cloud
  * Aggregate data on the edge. If data is coming from machine at a higher rate than you want, you can use ASA on the Edge to do time based aggregations of that data
* Azure Machine Learning modules to do real-time predition of machine failures or anomaly detection.
