Understanding how an Fprime packet is represented as bit is actually rather complicated, and spread across
a variety of places within the codebase.

It is best to think of a packet as being built from up three layers.
From outside to inside, those layers are:
* Framing
* ComPacket
* Packet type Specific

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

## Deframing
Note, that the default framing implementation only supports receiving two types of ComPackets.  `FW_PACKET_FILE` and `FW_PACKET_COMMAND` See Svc/Deframer/docs/sdd.md#L53

# ComPacket
In the default `Ref` application, `ComPacket` are used to create the `ComBuffer` that is passed to the `Framer`.

There are 7 types of `ComPacket`.  You can see the types listed in `ComPacket.hpp`

Unfortunately, there is no centralized code or common pattern for how these different types of packets are serialized.  Instead, it is distributed between packets that extend ComPacket (FW_PACKET_COMMAND, FW_PACKET_TELEM, FW_PACKET_LOG),
types that are used directly by the framer (FW_PACKET_FILE), types used by other classes that don't extend ComPacket (FW_PACKET_PACKETIZED_TLM), and types that seem to be unused (FW_PACKET_IDLE, FW_PACKET_UNKNOWN)

# List of packet structures
| ComPacket type           | class that extends ComPacket that uses it      | Other class that uses it to create something | is actually used in Ref application                                                                         | Inner packet structure                                                    |
|--------------------------|------------------------------------------------|----------------------------------------------|-------------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------|
| FW_PACKET_COMMAND        | CmdPacket.cpp                                  |                                              | yes                                                                                                         |                                                                           |
| FW_PACKET_TELEM          | TlmPacket.cpp                                  |                                              | yes                                                                                                         |                                                                           | 
| FW_PACKET_LOG            | LogPacket.cpp                                  |                                              | yes                                                                                                         |                                                                           |
| FW_PACKET_FILE           |                                                | Framer.cpp, GroundInterfaceComponentImpl.cpp | yes                                                                                                         | `FPRIME_START_WORD _ length _ FW_PACKET_FILE _ generic buffer _ checksum` |
| FW_PACKET_PACKETIZED_TLM | TlmPacketizer.cpp                              | no                                           | `FPRIME_START_WORD _ length _ FW_PACKET_LOG _ time _ log buffer   _ checksum`                               |
| FW_PACKET_IDLE           | Unused                                         | no                                           | `FPRIME_START_WORD _ length _ FW_PACKET_TELEM _  telemetry channel id _ time _ telemetry buffer _ checksum` |
| FW_PACKET_UNKNOWN        | ComPacket.cpp (default for m_type), Framer.cpp | yes                                          |


There are also some other packet classes that extend ComPacket but do not have their own type
* AmpcsEvrLogPacket

FW_PACKET_COMMAND, // !< Command packet type - incoming
                FW_PACKET_TELEM, // !< Telemetry packet type - outgoing
                FW_PACKET_LOG, // !< Log type - outgoing
                FW_PACKET_FILE, // !< File type - incoming and outgoing
                FW_PACKET_PACKETIZED_TLM, // !< Packetized telemetry packet type
                FW_PACKET_IDLE, // !< Idle packet
                FW_PACKET_UNKNOWN