
# Carapp Communication Protocol

## Wiring Diagram
![wiring diagram with SPI and serial](https://imgur.com/roMaYjj.png)

## SPI and Serial

For communication between the Lattepanda and the Raspberry Pi Pico, I chose to use these two physical mediums: **SPI and Serial**. 

The reasoning behind why I chose these two specific mediums is pretty simple. For one, communication with [the specific MCP2515 CAN module](https://www.aliexpress.us/item/3256804756901898.html) I got requires SPI. However, we don't have to worry about **how** to interface with the module over SPI because we can use this handy-dandy library [here](https://github.com/autowp/arduino-mcp2515).

As for why I chose serial, I initially planned on using a microcontroller interfacing library called [Firmata](https://github.com/firmata/protocol). However, due to the lack of SPI and fully fledged C++ support, I decided against using it. Instead, I'm planning on using the protocol defined in this doc to communicate over serial! 

(Do note that the Arduino Leonardo will only really act as a "forwarder" for any messages that it receives for protocol 2)

Which leads me to....

## The Protocol

As you might have noticed in the diagram, there are actually two different protocols that I'll be using. One is for communication to the Lattepanda's Arduino Leonardo and one is for communication to the Raspberry Pi Pico.

However, before we define both protocols, let's define a messaging structure.

### Messaging Structure

So, while I was looking into Firmata, I stumbled upon the messaging structure they were using for most of their communication. It appears that they ripped off [the MIDI SysEx messaging protocol](http://www.2writers.com/eddie/tutsysex.htm). So, I decided to do the same thing and rip-off the same protocol.

Each message has 9 fields. Here's an example in hex:

| 1 | 2 | 3| 4 | 5 | 6 | 7 | 8 | 9 |
|--|--|--|--|--|--|--|--|--|
| F0 | 41 | 01 | 00 | 12 | 00 7A | 40 00 7F | 47 | F7 |

Let's break this example message down by each field:

**[1]** - MIDI SysEx Specification, always one byte and contains `F0h`
**[2]** - MIDI SysEx Specification, always one byte and contains `41h`
**[3]** - Message target device ID, each device  has it's own unique ID. This can be from `00h` to `7Fh` meaning there are 127 possible unique devices.
**[4]** - Message protocol, this defines which protocol the message is using. This is important as it determines what data goes in to field 6.
**[5]** - Message type, this byte defines what the purpose of the message is. There are two possible values: `12h` and `11h`. `12h` should be used when the message is **sending/writing** information. `11h` should be used when the message is **requesting/reading** information.  Note that when sending/writing information, there won't be a reply even when the write is successful. Only when requesting/reading information will there be a reply in the form of a `12h` message.
**[6]** - Message address, this defines which "address" the message is referring to when it asks to read/write. This means that in the example, we are writing to address `7Ah` which could point to a variable. Each protocol will have it's own table that defines what variable is at each address. **This field always contains two bytes.** Note that when 
**[7]** - Message data, if field 5 is `12h`, then this is where the actual data that is being written is stored. If field 5 is `11h`, then this defines how many bytes the device expects to be sent back. This field can contain any amount of bytes.
**[8]** - Rolland checksum, used for error checking. Implementation is pretty simple--you can find more about it in the MIDI SysEx protocol page that I linked above but the gist is that it's calculated by adding up all the decimal values in fields 6 and 7  and then taking the reminder of that sum when divided by 128 (% operator) and subtracting that remainder from 128. So, it would look something like this for the example message:

	(00h + 7Ah + 40h + 00h + 7Fh) -> 0 + 122 + 64 + 0 + 127 = 313
	
	128 - (313%128) = 71

	71 -> 47h

**[9]** - MIDI SysEx Specification, always one byte and contains `F7h`

Ok so now that we know how the messaging works, let's define our protocols.

### Protocol 1 [LattePanda - Arduino Leonardo]

As mentioned above, note that any protocol 2 message sent to the Arduino Leonardo will be forwarded to the Raspberry Pi Pico and any messages sent from the Raspberry Pi Pico via a protocol 2 message will be forwarded back to the LattePanda.

Otherwise, here's the table to define message addresses for protocol 1.

|Function/variable|Message Address (hex)|Read/Write support|Data Type|# of Bytes|
|--|--|--|--|--|
|analogRead| 1A 01 | Read Only |`uint8_t`|1|
|analogWrite| 1A 02 | Write Only |`uint8_t`|1|
|digitalRead| 1A 04 | Read Only |`bool`|1|
|digitalWrite| 1A 05 | Write Only |`bool`|1|
|pinMode| 1A 06 | Write Only |`size_t`|2|

### Protocol 2 [LattePanda - Raspberry Pi Pico]

Here's the table for protocol 2.

|Function/variable|Message Address (hex)|Read/Write support|Data Type|# of Bytes|Notes|
|--|--|--|--|--|--|
|acSystemStatus| 1A 01 | Read Only |`uint8_t`|1|Struct packed with msgpack, struct defined below|

#### acSystemStatus msgpack C++ struct

	struct s{
	}