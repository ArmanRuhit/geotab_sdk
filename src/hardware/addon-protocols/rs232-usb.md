---
layout: page
permalink: /hardware/addon-protocols/rs232-usb/
title: Add-On Protocol - RS232 & USB
---

External devices can communicate with the Geotab GO device through the Third-Party RS232 and USB protocol below. Two-way communication is supported, allowing a MyGeotab API call to produce messages from the IOX device to reach the external device. The hardware interface will be one of the following:

 - [IOX-RS232 F/M](https://www.geotab.com/documentation/iox-rs232/ "IOX-RS232 Support Documentation")
 - [IOX-USB](https://www.geotab.com/documentation/iox-usb/ "IOX-USB Support Documentation")

## Special Requirements
### Enabling IOX-USB Data Transfer

To enable third-party data communication on the IOX-USB, apply the following custom parameter to the GO device through MyGeotab.

```xml
<GoParameters><Parameter Description="Enable USB Data" Offset="164" Bytes="02"/></GoParameters>
```

\* *Note* \- The GO device will automatically upgrade to the ProPlus rate plan once third-party data transfer begins.
### IOX-USB Communication Consideration
 
The IOX-USB operates as a USB 2.0 full-speed host. The maximum data transfer rate is 12 Mbit/s. The IOX-USB can use two methods to enumerate a USB device:

1. The [Android Open Accessory protocol (*AOA*)](https://source.android.com/devices/accessories/protocol.html). This [sample project](https://github.com/Geotab/android-external-device-example "Android Open Accessory Sample") can be used as a framework.
2. USB-CDC (Communications Device Class)

### Powering a device using the IOX-RS232 and IOX-USB
Both the IOX-USB and the IOX-RS232 can provide power to an Add-On Device. 

- The IOX-USB can provide 1.5A at 5V as a power output. 
- The IOX-RS232 supports 900mA at 12/24V to the external red (power) and black (ground) wires. However, it is not required to power the Add On device using the IOX-RS232.

### Grounding a device
Even if the Hardware Add-On has a separate connection to vehicle power and ground, it is still recommended to connect the Add-On ground to the ground wire of the IOX-RS232 as this will improve signal integrity.

### Serial Port Settings For Add-Ons

Geotab recommends that RS232/USB serial ports are programmed in accordance with the following specifications:

- Baud Rate: 9600 or 115200, Note: the device is equipped with autobaud detection so other standard rates are acceptable
- Parity: None
- Stop Bits: 1
- Flow Control: None

## Integration Process

The following process should be followed when integrating a third-party device with the GO device using our Third-Party Data Protocol.

### Contact Solutions Engineering

Contact the [Geotab Solutions Engineering team](mailto:soleng@geotab.com) with a detailed integration proposal, this should include:

 - A name for the integration
 - The interfacing hardware
 - Data types that will be sent to MyGeotab
 - The required Status Data
 - Direction of data transfer
 - Expected timelines for integrating
 
The Solutions Engineering team will respond with follow up questions to define the integration, and assign an External device ID, and any Status Data IDs that would be required. 

An additional resource is the [Hardware Integration Toolkit](https://docs.google.com/presentation/d/1nkmDYw2tscZxKaezFm5sR3jLItI3IRJTS6JIhgg0rFU/edit#slide=id.g625282e7fc_0_0) with integration walkthrough.

### Using Status Data IDs

There is an extensively defined Status Data ID list which can be found at [MyGeotab Diagnostics](https://docs.google.com/spreadsheets/d/1sy7IOhWmFoo40_I-ruOJO8bVTMLXqHa11d0oJtaoIcE/edit#gid=1133172080). Specifics regarding Status Data ID implementations can be found on the README sheet. 

### Handshake

An initial Handshake **is required** in order for the GO device to accept third-party data. Ignition must be on for the handshake process.

1. After powering up, the GO device will enter an external device detection cycle. The external device will be powered for 72 seconds. In this interval, the GO device will listen for a [Handshake Sync](#handshake-sync-auto-baud-detect-for-rs232) from the external device. The Handshake Sync is used to indicate that an external device is present. For implementations using the IOX-RS232, the Handshake Sync is also used to detect baud rate.
  - The external device must send the Handshake Sync message once per second.
  - If a Handshake Sync message is not detected from the external device after 72 seconds, the external device is powered down for 5 seconds, then powered up again to restart the detection cycle.
2. The GO device will reply to a Handshake Sync with a [Handshake Request](#msg-type-0x01-handshake-request).
3. The external device must reply with a [Handshake Confirmation](#msg-type-0x81-handshake-confirmation) message within 2 seconds. If the external device would like an acknowledgment from the GO device that it received the Handshake Confirmation message, the corresponding flag in the Handshake Confirmation message may be set.
4. After sending the Handshake Confirmation message, the external device can begin to send third-party data as required. For every [Third-Party Data Message](#msg-type-0x80-third-party-data-as-status-data) sent, the GO device will reply with a [Data Acknowledge message](#msg-type-0x02-third-party-data-acknowledge).
  - If the external device receives no response to a Third-Party Data message, it must restart the handshake process — returning to step 1 above.
5. The GO device may send a Handshake Request message at any time after the initial handshake. The external device must respond with a Handshake Confirmation message. If the external device does not respond, it must restart the handshake process — returning to step 1 above.

### Checksum

Each message will contain a 2-byte Fletcher's Checksum calculated across all the bytes of the message except the checksum itself. The checksum values are bytes, and as such overflow from 255 (0xFF) to 0 (0x00). The bytes used for the checksum calculation are all the bytes up to the checksum byte, including STX, LEN, TYPE, but not including ETX.

Checksum calculation pseudo code:

```js
byte ChkA = 0;
byte ChkB = 0;
// n is the number of bytes in the message
// up to, but not including, the checksum
for (i = 0; i < n; i++)
{
ChkA = ChkA + MsgBuffer[i];
ChkB = ChkB + ChkA;
}
// ChkA precedes ChkB in the message
```

### Data Endianness

All values must be sent using Little Endian Byte Order, meaning the least significant byte first.

## Messages from the GO device
### Msg Type 0x01: Handshake Request

Issued by GO device on receipt of the Handshake Sync and periodically re-sent to confirm that the external device is still attached.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x01 | 1 | 1 |
| Message Body Length = 0 | 1 | 2 |
| Checksum | 2 | 3 |
| ETX (0x03) | 1 | 5 |
| Reply: Handshake Confirmation ([Msg Type 0x81](#msg-type-0x81-handshake-confirmation)) |

### Msg Type 0x02: Third-Party Data Acknowledge

Issued by GO device on receipt of Third-Party Data from the External Device.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x02 | 1 | 1 |
| Message Body Length = 0 | 1 | 2 |
| Checksum | 2 | 3 |
| ETX (0x03) | 1 | 5 |

### Msg Type 0x21: GO Device Data

Issued by GO device every 2 seconds to a connected Enhanced HOS Device (ID: 4141) or periodically when a 0x85 request message is received.

- An Enhanced HOS Device must ACK this message with a 0x84 message.
- If the data is requested periodically using the 0x85 message, the ACK is optional.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x21 | 1 | 1 |
| Message Body Length >= 52 [1] | 1 | 2 |
| Date / Time [2] | 4 | 3 |
| Latitude | 4 | 7 |
| Longitude | 4 | 11 |
| Road Speed [3] | 1 | 15 |
| RPM | 2 | 16 |
| Odometer [4] | 4 | 18 |
| Status Flags (from LSB): <br> 1st bit: 1 = GPS Valid <br> 2nd bit: 1 = Ignition On <br> 3rd bit: 1 = Engine Bus Activity <br> 4th bit: 1 = Date/Time Valid <br> 5th bit: 1 = Speed From Engine <br> 6th bit: 1 = Odometer From Engine | 1 | 22 |
| Trip Odometer [4] | 4 | 23 |
| Total Engine Hours | 4 | 27 |
| Trip Duration [5] | 4 | 31 |
| GO Device ID [6] | 4 | 35 |
| Driver ID [7] | 4 | 39 |
| GO Device Serial Number | 12 | 43 |
| Checksum | 2 | Length + 3 |
| ETX (0x03) | 1 | Length + 5 |
| Reply: Device Data Ack ([Msg Type 0x84](#msg-type-0x84-device-data-ack)) |   |   |

1. All implementations of this message must cater for the message length increasing in the future.
2. "Date/Time" is a 'seconds' counter starting from 1st of January 2002.
3. If Road Speed from the engine is not available, GPS speed is used.
4. Increase of odometer since the most recent ignition on. If Odometer is not available, GPS device distance is used.
5. Time passed since the most recent ignition on.
6. GO Device ID is a legacy field. It will contain invalid values by April 15, 2021.
7. Driver ID only available when using the IOX-NFC.

#### *Conversions*

| **Data** | **Conversion** | **Units** |
| --- | --- | --- |
| Engine Road Speed | 1 | km/h |
| Odometer | 0.1 | km |
| RPM | 0.25 | RPM |
| Lat/Long | 1e-7 | degrees |
| Engine Hours | 0.1 | h |
| Trip Duration | 1 | s |

### Msg Type 0x22: Binary Data Response

Issued by the GO device on successful/unsuccessful transmission of Binary Data (via Msg Type 0x86) to the server.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x22 | 1 | 1 |
| Message Body Length = 4 | 1 | 2 |
| Binary data transmission success <br> 0 = transmission failure <br> 1 = transmission success | 1 | 3 |
| Reserved | 3 | 4 |
| Checksum | 2 | 7 |
| ETX (0x03) | 1 | 9 |

### Msg Type 0x23: Binary Data Packet

Issued by the GO device on receipt of Binary Data from the server destined for the external device. This message format will only be used if the corresponding "Binary Data Packet Wrapping" flag has been set by the external device during the Handshake Confirmation. The payload of the binary data packet message will be the raw bytes as sent from the server.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x23 | 1 | 1 |
| Message Body Length = x (0 - 249) | 1 | 2 |
| Binary Data | x | 3 |
| Checksum | 2 | 3+x |
| ETX (0x03) | 1 | 5+x |

## Messages from External Device

### Handshake Sync (Auto-BAUD detect for RS232)

Issued by External Device every second until the Handshake Request is received.

|   | Bytes | Position |
| --- | --- | --- |
| Sync Char (0x55) | 1 | 0 |
| Reply: Handshake Request ([Msg Type 0x01](#msg-type-0x01-handshake-request)) |

### Msg Type 0x81: Handshake Confirmation

Issued by the External Device when it receives the Handshake Request.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x81 | 1 | 1 |
| Message Body Length = 4 | 1 | 2 |
| External Device ID (assigned by Geotab) | 2 | 3 |
| Flags <br> 1st bit: Handshake Confirmation ACK <br> 2nd bit: Binary Data Packet Wrapping <br> All other bits:Reserved for future implementation, must be set to 0 | 2 | 5 |
| Checksum | 2 | 7 |
| ETX (0x03) | 1 | 9 |

Handshake Confirmation ACK:

- 0: No Ack to Handshake Confirmation message will be sent by the GO device.
- 1: The Handshake Confirmation is to be ACKed with a Third Party Data Acknowledge message.

Binary Data Packet Wrapping:

- 0: The passthrough data from the server will be passed to the external device without modification.
- 1: The passthrough data from the server will be wrapped in a Binary Data Packet message before being sent to the external device.

### Msg Type 0x80: Third-Party Data as Status Data

Issued by the external device whenever it requires Third-Party Data to be saved on the GO device as Status Data.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x80 | 1 | 1 |
| Message Body Length = 6 | 1 | 2 |
| Data ID (assigned by Geotab) | 2 | 3 |
| Data | 4 | 5 |
| Checksum | 2 | 9 |
| ETX (0x03) | 1 | 11 |
| Reply: Third-Party Data Ack ([Msg Type 0x02](#msg-type-0x02-third-party-data-acknowledge)) |

### Msg Type 0x82: Free Format Third-Party Data

Issued by the external device whenever it wants Third-Party Data to be saved on the GO device in a free format (1 to 27 bytes) that will be saved into MyGeotab as Custom Data. Rate limit is 500 logs per 10 minutes. If you exceed the rate limit, the GO device will stop taking data from the IOX.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x82 | 1 | 1 |
| Message Body Length = x (1 to 27) | 1 | 2 |
| Data | x | 3 |
| Checksum | 2 | 3 + x |
| ETX (0x03) | 1 | 5 + x |
| Reply: Third-Party Data Ack ([Msg Type 0x02](#msg-type-0x02-third-party-data-acknowledge)) |

### Msg Type 0x84: Device Data ACK

Issued by the External Device on receipt of the GO Device Data message.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x84 | 1 | 1 |
| Message Body Length = 0 | 1 | 2 |
| Checksum | 2 | 3 |
| ETX (0x03) | 1 | 5 |

For the purpose of acknowledging the GO Device Data message when connected as an Enhanced HOS Device:

- The GO device will keep streaming the GO Device Data messages even if no ACK is received for up to 30 seconds.
- If no ACK is received in that time frame the GO Device will send an External Device Disconnected record to the server and will wait for a new Handshake Sync request from the External Device.
- If the ACK message is received within the 30 seconds the counter will be re-initialized.

### Msg Type 0x85: Request Device Data Message

This is a request-response message. It can be issued by the External Device whenever it wishes to receive the Device Data Info Message (Msg Type 0x21).

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x85 | 1 | 1 |
| Message Body Length = 0 | 1 | 2 |
| Checksum | 2 | 3 |
| ETX (0x03) | 1 | 5 |
| Reply: GO Device Data ([Msg Type 0x21](#msg-type-0x21-go-device-data)) |   |   |

### Msg Type 0x86: Binary Data Packet

Sent by the external device when sending binary data directly to the server. The contents of the message will be ignored by the GO device and simply sent on to the server for processing. The GO device will respond with the Binary Data Response message indicating whether the data was successfully sent via the modem.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x86 | 1 | 1 |
| Message Body Length = x (0 - 250) | 1 | 2 |
| Binary Data | x | 3 |
| Checksum | 2 | 3+x |
| ETX (0x03) | 1 | 5+x |
| Reply: Binary Data Response ([Msg Type 0x22](#msg-type-0x22-binary-data-response)) |   |   |

The payload of the binary data needs to adhere to protocols understood by the Geotab servers. MIME protocol is one these protocols. Please see [Appendix C](#appendix-c-using-binary-data-messages-to-transfer-mime-data) for implementation details.

### Msg Type 0x87: Third-Party Data as Priority Status Data

Priority Status Data will follow an expedited processing workflow on the GoDevice but will otherwise be treated the same as the 0x80 Status Data message. It will also be logged using an Iridium modem connection if available.

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x87 | 1 | 1 |
| Message Body Length = 6 | 1 | 2 |
| Data ID (assigned by Geotab) | 2 | 3 |
| Data | 4 | 5 |
| Checksum | 2 | 9 |
| ETX (0x03) | 1 | 11 |
| Reply: Third-Party Data Ack ([Msg Type 0x02](#msg-type-0x02-third-party-data-acknowledge)) |

### Msg Type 0x89: Ping

After handshaking, this message can be issued periodically by the External Device to check that the GO device is active and ready. The GO device will normally reply with the Third-Party Data Ack (Msg Type 0x02). If this reply is not received, the External Device should reset and begin sending the Handshake Sync (0x55).

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x89 | 1 | 1 |
| Message Body Length = 0 | 1 | 2 |
| Checksum | 2 | 3 |
| ETX (0x03) | 1 | 5 |
| Reply: Third-Party Data Ack ([Msg Type 0x02](#msg-type-0x02-third-party-data-acknowledge)) |

## Messages from MyGeotab

To send messages from MyGeotab to the external device, please download the source code of the [Starter Kit](https://geotab.github.io/sdk/software/js-samples/#starter-kit) sample, and replace the [Sample API](https://github.com/Geotab/sdk/blob/master/src/software/js-samples/starterKit.html#L76) with the following script. The alternative is paste the script in the [Runner](https://geotab.github.io/sdk/software/api/runner.html).
```javascript
    api.call("Add", {
            "typeName": "TextMessage",
            "entity": {
              "user": {
                "id": user.id               //Replace with user id of interest
              },
              "device": {
                "id": device.id             //Replace with device id of interest
              },
              "messageContent": {
                "contentType": "SerialIox",
                "channel": 1,               //Taken from Get<IoxAddOn> call
                "data": base64_encoded_byte //Replace with your data encoded in base64
              },
              "isDirectionToVehicle": true
            }
          }, function(result) {
              console.log("Done: ", result);
          }, function(e) {
              console.error("Failed:", e);
          });)
```
To send MIME messages from MyGeotab to the external device, please use the following script instead:
```javascript
    api.call("Add", {
    "typeName": "TextMessage",
    "entity":{
        "user":{"id":user.id},                    //Replace with user id of interest
        "device":{"id":device.id},                //Replace with device id of interest
        "messageContent":{
            "contentType":"MimeContent",
            "channelNumber":1,
            "mimeType":"text",                    //Can be changed to any free format text value
			      "binaryDataPacketDelay":"00:00:03.0000000", //applies a 
            //configurable delay of up to 5 seconds in between each sequenced message
            // of a multimessage MIME payload
            "data":base64_encoded_byte //Replace with your data encoded in base64
        },
        "isDirectionToVehicle":true},
      }, function(result) {
          console.log("Done: ", result);
      }, function(e) {
          console.error("Failed:", e);
      });
```

## Appendices

### Appendix A: Raw Message Data Example for IOX-USB & IOX-RS232

```js
Handshake Sync from External Device
0x55... 0x55... 0x55

Handshake Request from GO device
0x02, 0x01, 0x00, 0x03, 0x08, 0x03

Handshake Confirmation from External Device (4208 is a test Device ID)
(Device ID: 4208 = 0x00001070)
0x02, 0x81, 0x04, 0x70, 0x10, 0x00, 0x00, 0x07, 0x18, 0x03

Third-Party Data from External Device
(Status Data ID: 35349 = 0x8A15, Data Value: 200 = 0x000000C8)
0x02, 0x80, 0x06, 0x15, 0x8A, 0xC8, 0x00, 0x00, 0x00, 0xEF, 0x8C, 0x03

Third-Party Data Acknowledge from GO device
0x02, 0x02, 0x00, 0x04, 0x0A, 0x03
```

### Appendix B: Sample Message Flow for IOX-USB & IOX-RS232

 ![]({{site.baseurl}}/hardware/addon-protocols/rs232-usb_0.png)

### Appendix C: Using Binary Data Messages to Transfer MIME Data

MIME-type data can be transferred from an external device to the server via the GO device. The Message Flow is similar to that outlined in [Appendix B](#appendix-b-sample-message-flow-for-iox-usb--iox-rs232), with the following variations:
1. Third-Party Data Message is instantiated as Binary Data Packet Containing MIME Type Data, whose format is [such](#binary-data-packets-containing-mime-type-data)
2. Data Acknowledge Message is instantiated as Binary Data Response (0x22)
3. After the last Binary Data Response, add a Binary Data Packet Containing MIME Type Acknowledge, whose format is [such](#binary-data-packet-containing-mime-type-acknowledge). Once the complete payload of the MIME message is successfully received by MyGeotab, a MIME ACK will be sent back to the GO device.

MIME-type data will be saved as a MIME-type blob on the server. The blob can be accessed through the software SDK as a [TextMessage](/software/api/reference/#TextMessage). The SDK can also be used to send MIME-type data from the server to an external device connected to a GO device.

### The MIME Type Protocol:

|   | Bytes | Position |
| --- | --- | --- |
| MIME type length = x | 1 | 0 |
| MIME type in ASCII | x | 1 |
| Payload Length = y | 4 | 1 + x |
| Binary Payload | y | 5 + x |

The MIME protocol will be an inner protocol within the binary data packet messages. The MIME data will be broken into 250 byte chunks and sent within binary data packet messages. The first byte within the message will be a sequence counter; all remaining bytes will contain the MIME data.

### Binary Data Packets Containing MIME Type Data

This is an example of binary data packets for image data transferred using the MIME type "image/jpeg". The image size is 83000 bytes.

**First packet:**

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x86 | 1 | 1 |
| Message Body Length = 250 | 1 | 2 |
| **Sequence number = 0** | **1** | **3** |
| **MIME type length = 10** | **1** | **4** |
| **MIME type ("image/jpeg")** | **10** | **5** |
| **Payload Length = 83000** | **4** | **15** |
| **Binary Payload (the first 234 bytes)** | **234** | **19** |
| Checksum | 2 | 253 |
| ETX (0x03) | 1 | 255 |

**Second packet:**

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x86 | 1 | 1 |
| Message Body Length = 250 | 1 | 2 |
| **Sequence number = 1**\* | **1** | **3** |
| **Binary Payload (the next 249 bytes)** | **249** | **4** |
| Checksum | 2 | 253 |
| ETX (0x03) | 1 | 255 |

\* If the sequence number reaches 255 (0xFF) and more packets need to be sent, the sequence number must reset to a value of 1 and continue counting. **A sequence number of 0 must only be used for the first packet.**

### Binary Data Packet Containing MIME Type Acknowledge

|   | Bytes | Position |
| --- | --- | --- |
| STX (0x02) | 1 | 0 |
| Message Type = 0x23 | 1 | 1 |
| Message Body Length = 9+x | 1 | 2 |
| Sequence Number = 0 | 1 | 3 |
| MIME type length = 3 | 1 | 4 |
| MIME type in ASCII = 'ACK' | 3 | 5 |
| Payload Length = x | 4 | 8 |
| Total Number of Payload Bytes Received | x | 12 |
| Checksum | 2 | 12+x |
| ETX (0x03) | 1 | 14+x |
