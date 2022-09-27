Understanding how an Fprime packet is represented as bit is actually rather complicated, and spread across
a variety of places within the codebase.

It is best to think of a packet as being built from up three layers.
From outside to inside, those layers are:
* Framing
* ComPacket
* Packet Specific

# Framing
Framing is the process of wrapping a packet to prepare it for transmission to the ground.
This is possible to customize by implementing a different `FramingProtocol`
* 
* Documentation:
  * https://github.com/nasa/fprime/blob/devel/Svc/Framer/docs/sdd.md
  * https://github.com/nasa/fprime/blob/master/Svc/FramingProtocol/docs/sdd.md
* Default Implementation: `FPrimeFraming::frame` (https://github.com/nasa/fprime/blob/master/Svc/FramingProtocol/FprimeProtocol.cpp#L24)

The default FPrimeFraming implementation outputs the following structure
```
FPRIME_START_WORD | length of inner data plus packet_type | packet_type | inner data (without length) | checksum
```

The framer has to handle two different types of inputs though: a `ComBuffer` (through the `comIn` port)
and a generic buffer (through the `bufferIn` port).
It achieves this by using the `ComPacket::FW_PACKET_FILE` packet type to handle the generic buffer,
and relying on the ComBuffer to contain its packet type information.

This leads to somewhat confusing code interactions between the `Framer` class and the `FprimeFraming` implementation of the framing protocol.  The `FprimeFraming` implementation 
uses the packet type `FW_PACKET_UNKNOWN` to signal that it has been passed a `ComBuffer`.  In theory it supports being passed any other packet type, but in the actual usage by 
the `Framer`, the only other type it will ever be passed is `FW_PACKET_FILE`.

## Deframing
Note, that the default framing implementation only supports receiving two types of ComPackets.  `FW_PACKET_FILE` and `FW_PACKET_COMMAND` See Svc/Deframer/docs/sdd.md#L53

# ComPacket
In the default `Ref` application, `ComPacket` are used to create the `ComBuffer` that is passed to the `Framer`.

There are 7 types of `ComPacket` specifid in the `ComPacketType` enum in `ComPacket.hpp`

Unfortunately, there is no centralized code or common pattern for how each these different types are used to serialize data.  There are five different approaches.
* Some types (FW_PACKET_COMMAND, FW_PACKET_TELEM, FW_PACKET_LOG) are used by classes that extend ComPacket (CmdPacket.cpp, TlmPacket.cpp, LogPacket.cpp)
* One type (FW_PACKET_FILE) is used directly by the framer (Framer.cpp) and a class that doesn't extend ComPacket (GroundInterfaceComponentImpl.cpp)
* One type (FW_PACKET_PACKETIZED_TLM) is used by a different class that doesn't extend ComPacket (TlmPacketizer.cpp)
* One type (FW_PACKET_UNKNOWN) that is unused for serialization, but is used for interal logic as mentioned in the "Framing" section above.
* One type (FW_PACKET_IDLE) that is completely unused

## Other things to note

### FW_PACKET_HAND
There is also an additional packet type `FW_PACKET_HAND` that is present in the `fprime-gds` ground software code but not in the `fprime` flight code.

### Other ComPacket classes without custom types

There are also some other packet classes that extend ComPacket but do not have their own type.

* AmpcsEvrLogPacket has its own custom binary format which does not match any of the other packet types

# Packet specific
## FW_PACKET_COMMAND
There are two serialized pieces of a command packet
* op code: 
* arguments: This is the binary serialized form of each argument to that command, concatenated together in order

Interestingly, there is no serialization logic for command packets in the `fprime` flight code, since command packets are only received by flight, not sent.

## FW_PACKET_TELEM
A telemetry packet has three things serialized

* telemetry channel id: 
* time tag: 
* telemetry buffer: 

Note: `FW_PACKET_TELEM` has special logic based on the compile time macro `FW_AMPCS_COMPATIBLE` that changes its binary representation.  We can
read [configuring-fprime.md](../UsersGuide/dev/configuring-fprime.md) to understand its usage.  The default setting (`FW_AMPCS_COMPATIBLE=False` is what is shown in the table below)

## FW_PACKET_LOG
* time tag:
* log buffer

## FW_PACKET_FILE
* time tag:
* generic buffer


## FW_PACKET_PACKETIZED_TLM
The code that generates `FW_PACKET_PACKETIZED_TLM` is very difficult to follow and needs more work to clearly document it.

## FW_PACKET_PACKETIZED_TLM
This packet type is unused in the flight code, so there is not structure for it

## FW_PACKET_UNKNOWN
This packet type is used for internal communication (and not serialied for the ground), so it is unknown what its internal format would look like

# Summary
This table summarizes the above information, and includes the specific inner format used for each ComPacketType
| ComPacketType            | class that extends ComPacket that uses it | Other class that uses it                                       | is actually used in Ref application   | Binary packet structure                                                                                         |
|--------------------------|-------------------------------------------|----------------------------------------------------------------|---------------------------------------|-----------------------------------------------------------------------------------------------------------------|
| FW_PACKET_COMMAND        | CmdPacket.cpp                             |                                                                | yes                                   | `FPRIME_START_WORD _ length _ FW_PACKET_COMMAND _ op code _ arg buffer _ checksum`                              |
| FW_PACKET_TELEM          | TlmPacket.cpp                             |                                                                | yes                                   | `FPRIME_START_WORD _ length _ FW_PACKET_TELEM _  telemetry channel id _ time tag _ telemetry buffer _ checksum` | 
| FW_PACKET_LOG            | LogPacket.cpp                             |                                                                | yes                                   | `FPRIME_START_WORD _ length _ FW_PACKET_LOG _  log id _ time tag _ log buffer _ checksum`                       |
| FW_PACKET_FILE           |                                           | Framer.cpp, GroundInterfaceComponentImpl.cpp (see notes below) | yes                                   | `FPRIME_START_WORD _ length _ FW_PACKET_FILE _ generic buffer _ checksum`                                       |
| FW_PACKET_PACKETIZED_TLM |                                           | TlmPacketizer.cpp                                              | no                                    | ???                                                                                                             |
| FW_PACKET_IDLE           |                                           |                                                                | no                                    | ???                                                                                                             |
| FW_PACKET_UNKNOWN        | ComPacket.cpp (default for m_type)        |                                                                | yes                                   | `FPRIME_START_WORD _ length _ ??? _ checksum`                                                                   |


# Other notes


## FW_PACKET_FILE and GroundInterfaceComponentImpl
The class `GroundInterfaceComponentImpl` implements a slightly different framing logic that the Framer.  It does not use a checksum, but instead uses a `END_WORD`.
`FPRIME_START_WORD _ length _ FW_PACKET_FILE _ generic buffer _ END_WORD`


# Open questions
## About the fprime code
* Why does `GroundInterfaceComponentImpl::frame_send` re-implement framing, instead of using the FPrimeFraming?
* How else are FW_PACKET_FILE objects given to the framer?  Does they go through the generic buffer port?  Why not have FilePacket act like the other packet classes???
* In FprimeFraming::frame, the last line is `m_interface->send(buffer);`.  Does this modify the outgoing
data in any way, for example adding the length?

## About my documentation
* Best way to include source code links in this documentation