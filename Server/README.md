# xbee-api [![Build Status](https://secure.travis-ci.org/jouz/xbee-api.png?branch=master)](http://travis-ci.org/jouz/xbee-api)

[Digi's xbee modules](http://www.digi.com/xbee) are good for quickly building low power wireless networks. They can be used to send/receive text data from serial ports of different devices. XBees can also be used alone for their on board digital and analog I/O capabilities.

**xbee-api** helps you parsing and building API frames that are used to communicate with XBee modules. **xbee-api** does *not* take care of the serial connection itself, but it is easy to hook it up to modules such as [serialport](https://github.com/voodootikigod/node-serialport/).

Note that higher-level abstraction as currently provided in [svd-xbee](https://github.com/jouz/svd-xbee/) is not part of this module anymore, but will be factored out to third modules (see [xbee-stream](https://github.com/jouz/xbee-stream/) and [xbee-stream-nodes](https://github.com/jouz/xbee-stream-nodes/), WiP).

## GETTING STARTED
Install the module with: `npm install xbee-api`

```javascript
var xbee_api = require('xbee-api');
var C = xbee_api.constants;
var xbeeAPI = new xbee_api.XBeeAPI();

// Something we might want to send to an XBee...
var frame_obj = {
  type: C.FRAME_TYPE.AT_COMMAND,
  command: "NI",
  commandParameter: [],
};
console.log(xbeeAPI.buildFrame(frame_obj));
// <Buffer 7e 00 04 08 01 4e 49 5f>


// Something we might receive from an XBee...
var raw_frame = new Buffer([
	0x7E, 0x00, 0x13, 0x97, 0x55, 0x00, 0x13, 0xA2, 0x00, 0x40, 0x52, 0x2B,
	0xAA, 0x7D, 0x84, 0x53, 0x4C, 0x00, 0x40, 0x52, 0x2B, 0xAA, 0xF0
]);
console.log(xbeeAPI.parseFrame(raw_frame));
// { type: 151,
//   id: 85,
//   remote64: '0013a20040522baa',
//   remote16: '7d84',
//   command: 'SL',
//   commandStatus: 0,
//   commandData: [ 64, 82, 43, 170 ] }
```

## DOCUMENTATION

### The XBeeAPI Class

To get an instance of the `xbeeAPI` of the `XBeeAPI` class:
```javascript
var xbee_api = require('xbee-api');
var xbeeAPI = new xbee_api.XBeeAPI({
	// default options:
	api_mode: 1
});
```
In the following a documentation of all class methods.

#### xbeeAPI.buildFrame(frame)
Returns an API frame (buffer) created from the passed frame object. See ***Creating frames from objects to write to the XBee*** for details on how these passed objects are specified.

#### xbeeAPI.parseFrame(buffer)
Parses and returns a frame object from the buffer passed. Note that the buffer must be a complete frame, starting with the start byte and ending with the checksum byte. See ***Objects created from received API Frames*** for details on how the retured objects are specified.

#### xbeeAPI.rawParser()
Returns a parser function with the profile `function(emitter, buffer) {}`. This can be passed to a serial reader such as serialport. Note that XBeeAPI will not use the emitter to emit a parsed frame, but it's own emitter (see Event: 'frame_object').

#### xbeeAPI.parseRaw(buffer)
Parses data in the buffer, assumes it is comming directly from the XBee. If a complete frame is collected, it is emitted as Event: 'frame_object'.

#### Event: 'frame_object'
Is emitted whenever a complete frame is collected and parsed.

#### xbeeAPI.nextFrameId()
Increments the internal `frameId` counter and returns it. Useful for building requests where we want to identify the respective response frame later on.

### Creating frames from objects to write to the XBee
There are four basic requests we can make to the XBee of which two are essentially the same. We can make (queued) command requests (0x08 and 0x09) and *remote* command request (0x17). These all behave pretty much the same as writing AT commands in AT mode. Lastly, there are transmit requests which can be used to send your own data to other devices.

#### 0x08: AT Command Request
```javascript
{
	type: 0x08,
	id: 0x52, // optional, nextFrameId() is called per default
	command: "NJ",
	commandParameter: [],
}
```
Execute the AT command set in `command`, optionally set a `comandParameter` value. An empty parameter usually queries the AT parameter value.

#### 0x09: AT Command Queue Request
```javascript
{
	type: 0x09,
	id: 0x01, // optional, nextFrameId() is called per default
	command: "BD",
	commandParameter: [ 0x07 ]
}
```
Pretty much the same as AT Command Requests, except that the commands are queued and applied at once when either an `AC` command is queued or a regular AT command request is sent.

#### 0x17: AT Remote Command Request
```javascript
{
	type: 0x17,
	id: 0x01, // optional, nextFrameId() is called per default
	destination64: "0013a20040401122",
	destination16: "fffe", // optional, "fffe" is default
	remoteCommandOptions: 0x02, // optional, 0x02 is default
	command: "BH",
	commandParameter: [ 0x01 ] // Can either be string or byte array.
}
```
Behaves just as AT Command Requests, with additional `destination64/16` parameters set to the remote node on which the AT command is to be executed.

#### 0x10: Transmit Request
```javascript
{
	type: 0x10,
	id: 0x01, // optional, nextFrameId() is called per default
	destination64: "0013a200400a0127",
	destination16: "fffe", // optional, "fffe" is default
	broadcastRadius: 0x00, // optional, 0x00 is default
	options: 0x00, // optional, 0x00 is default
	data: "TxData0A" // Can either be string or byte array.
}
```
Transmit your own `data` to a remote node.

### Objects created from received API Frames
Objects created from API frames that the XBee would recieve contain a `type` property that identifies the frame type. If the frame is a response to a query made earlier, the `id` that was used for that request is also included.

#### 0x88: AT Command Response
```javascript
{
	type: 0x88,
	id: 0x01,
	command: "BD",
	commandStatus: 0x00,
	commandData: []
}
```
This is a response to a AT command request, for example to query or change an AT parameter value on the XBee module. The command was, in this case, setting the `BD` parameter of module. The command status `0` means `OK` (see [Constants]() for more), which means that the baud rate was changed successfully. 

#### 0x97: AT Remote Command Response
```javascript
{
	type: 0x97,
	id: 0x01,
	remote64: "0013a20040522baa",
	remote16: "7d84",
	command: "SL",
	commandStatus: 0x00,
	commandData: [ 0x40, 0x52, 0x2b, 0xaa ]
}
```
This is a response to a *remote* AT command request, for example to query or change an AT parameter value on another device in the network. This seems to be a response from the node with the address `0013a20040522baa`. The requested command was, in this case, `SL`. The command status `0` means `OK`.


#### 0x8b: Transmit Status
```javascript
{
	type: 0x8b,
	id: 0x01,
	remote16: "7d84",
	transmitRetryCount: 0,
	deliveryStatus: 0,
	discoveryStatus: 1
}
```
This status is received after sending out a transmit request to the XBee (i.e. to send some text data to another module). The status contains the 16bit network address `remote16`, the number of transmission retries, information about whether delivery was successful and about any discoveries made.

#### 0x8a: Modem Status
```javascript
{
	type: 0x8a,
	modemStatus: 0x06
}
```
These statuses give information about the general operation of the XBee. See the [Constants]() section for more.

#### 0x90: Receive Packet
```javascript
{
	type: 0x90,
	remote64: "0013a20040522baa",
	remote16: "7d84",
	receiveOptions: 0x01,
	data: [ 0x52, 0x78, 0x44, 0x61, 0x74, 0x61 ]
}
```
This frame contains general data (such as text data) received from remote nodes.

#### 0x92: ZigBee IO Data Sample Rx
```javascript
{
	type: 0x92,
	remote64: "0013a20040522baa",
	remote16: "7d84",
	receiveOptions: 0x01,
	numSamples: 1,
	digitalSamples: {
		"DIO2": 1,
        "DIO3": 0,
        "DIO4": 1
	},
	analogSamples: {
		"AD1": 644
	}
}
```
An I/O data sample that contains information about the state of the digital and analog I/O pins that are set to sample data. Here, pins `DIO2` & `DIO4` read `HIGH`, `DIO3` reads `LOW`, and `AD1` samples an analog voltage of `644mV`.

#### 0x95: Node Identification Indicator
```javascript
{
	type: 0x95,
	sender64: "0013a20040522baa"
	sender16: "7d84"
	receiveOptions: 0x02
	remote64: "0013a20040522baa"
	remote16:"7d84"
	nodeIdentifier: "MY_ROUTER",
	remoteParent16: "fffe",
	deviceType: 0x01,
	sourceEvent: 0x01
}
```
Modules with the `JN` (Join Notification) parameter enabled will transmit a broadcast Node Identification Indicator packet on power up and when joining. This can also be sent when the D0 button is pressed. Which of these events occurred is set in the `sourceEvent` property. `sender64/16` here is the one from who the packet was received, whereas `remote64/16` is the identified node itself (may be the same).

### Constants
You don't have to remember the hex-numbers of the frame types, command options, status types, etc. Everything is defined in two-way constants. See the examples below:

```javascript
var xbee_api = require('xbee-api');
var C = xbee_api.constants;

// Frame types (frame.type):
C.FRAME_TYPE.ZIGBEE_TRANSMIT_REQUEST; // 0x10
C.FRAME_TYPE[0x10] // "ZigBee Transmit Request (0x10)";

// Command Status (frame.commandStatus)
C.COMMAND_STATUS.ERROR; // 0x01
C.COMMAND_STATUS[0x01]; // "(Error (0x01)"

// Discovery Status (frame.discoveryStatus)
C.DISCOVERY_STATUS.ADDRESS_DISCOVERY // 0x01
C.DISCOVERY_STATUS[0x01] // "Address Discovery (0x01)"

// Delivery Status (frame.deliveryStatus)
C.DELIVERY_STATUS.ADDRESS_NOT_FOUND // 0x24
C.DELIVERY_STATUS[0x24] // "Address Not Found (0x24)"

// Modem Status (frame.modemStatus)
C.MODEM_STATUS.JOINED_NETWORK // 0x02
C.MODEM_STATUS[0x02] // "Joined Network (0x02)"

// Receive Options (frame.receiveOptions)
C.RECEIVE_OPTIONS.PACKET_ACKNOWLEDGED // 0x01;
C.RECEIVE_OPTIONS[0x01] // "Packet Acknowledged (0x01)"

// Device Type (frame.deviceType)
C.DEVICE_TYPE.END_DEVICE // 0x02
C.DEVICE_TYPE[0x02] // "End Device (0x02)"
```

Please refer to `lib/constants.js` for a complete list, and the module documentation [here](http://ftp1.digi.com/support/documentation/90000976_M.pdf "http://ftp1.digi.com/support/documentation/90000976_M.pdf") for more explanation.

## EXAMPLES
To combine with [serialport](https://github.com/voodootikigod/node-serialport/), we use the rawParser(). Make sure to set your baudrate, AP mode and COM port. 
```javascript
var util = require('util');
var SerialPort = require('serialport').SerialPort;
var xbee_api = require('xbee-api.js');

var C = xbee_api.constants;

var xbeeAPI = new xbee_api.XBeeAPI({
  api_mode: 1
});

var serialport = new SerialPort("COM19", {
  baudrate: 57600,
  parser: xbeeAPI.rawParser()
});

serialport.on("open", function() {
  var frame_obj = { // AT Request to be sent to 
    type: C.FRAME_TYPE.AT_COMMAND,
    command: "NI",
    commandParameter: [],
  };

  serialport.write(xbeeAPI.buildFrame(frame_obj));
});

// All frames parsed by the XBee will be emitted here
xbeeAPI.on("frame_object", function(frame) {
	console.log(">>", frame);
});
```

To link a received frame object to a request we earlier sent, we have to set and remember the `frame.id` of our request. Then, when a new frame object is emitted, we could look it up and route the response back.

```javascript

var frameId = xbee.nextFrameId();
var frame_obj = {
	type: C.FRAME_TYPE.AT_COMMAND,
	id: frameId,
	command: "NI",
	commandParameter: []
};

serialport.write(xbeeAPI.buildFrame(frame_obj));

// All frames parsed by the XBee will be emitted here
xbeeAPI.on("frame_object", function(frame) {
	if (frame.id == frameId &&
	    frame.type == C.FRAME_TYPE.AT_COMMAND_RESPONSE) {
		// This frame is definitely the response!
		console.log("Node identifier:",
			String.fromCharCode(frame.commandData));
	} else {
		// This is some other frame
	}
});
```

See the [examples folder](https://github.com/jouz/xbee-api/tree/master/examples) in the repository for more examples.

## CONTRIBUTE
Feel free to send a pull request. There are nodeunit test in the `test/` folder.

## SUPPORTED XBEE MODELS
Both ZNet 2.5 and ZIGBEE modules should be supported. Since ZIGBEE offers more features and is more robust, you might be interested in upgrading your modules from ZNet 2.5 to ZIGBEE: [upgradingfromznettozb.pdf](ftp://ftp1.digi.com/support/documentation/upgradingfromznettozb.pdf).  
Development is done using Series 2 XBee modules with XB24-ZB (ZIGBEE) firmware. In specific, this document is used as reference: [90000976_M.pdf](http://ftp1.digi.com/support/documentation/90000976_M.pdf "http://ftp1.digi.com/support/documentation/90000976_M.pdf").

Modules must run in API mode. Both AP=1 and AP=2 modes are supported.

## LICENSE
Copyright (c) 2013 Jan Kolkmeier  
Licensed under the MIT license.
