Understanding how an Fprime packet is represented as bit is actually rather complicated, and spread across
a variety of places within the codebase.

It is best to think of a packet as being built from up three layers.
From outside to inside, those layers are:
* Framing
* Common Packet
* Packet Specific

# Framing
Framing is the process of wrapping a packet to prepare it for transmission to the ground.
This is possible to customize by implementing a different `FramingProtocol`
* 
* Documentation:
  * https://github.com/nasa/fprime/blob/devel/Svc/Framer/docs/sdd.md
  * https://github.com/nasa/fprime/blob/master/Svc/FramingProtocol/docs/sdd.md
* Default Implementation: `FPrimeFraming::frame` (https://github.com/nasa/fprime/blob/master/Svc/FramingProtocol/FprimeProtocol.cpp#L24)

The default FPrimeFraming ensures that its output always follows the structure
```
FPRIME_START_WORD | length of inner data plus packet_type | packet_type | inner data (without length) | checksum
```

The framer has to handle two different types of inputs though: a ComBuffer (through the `comIn` port)
and a generic buffer (through the `bufferIn` port).
It achieves this by using the `ComPacket::FW_PACKET_FILE` packet type to handle the generic buffer,
and relying on the ComBuffer to contain its packet type information.

# Common packet
The `ComBuffer` is just a way for the different FPrime components to pass around in-memory buffers.  To actually
understand the structure inside a `ComBuffer`, we need to look at the different components that are connected to the 
`comIn` port of the Framer.

This will unfortunately depend on how you're structured your FPrime deployment.  It's controlled through either your FPP
or xml topology.

In the `Ref` application, only two components are connected to the `comIn` port of the framer,
`chanTlm`, and `eventLogger`.

## chanTlm
The `chanTlm` uses `Fw::TlmPacket` to create its `ComBuffer`, so it looks like
```
type (FW_PACKET_TELEM) | telemetry channel id | time | telemetry buffer
```
(Note: The inclusion of `FW_PACKET_TELEM` and `time` depend on the setting of the `FW_AMPCS_COMPATIBLE` macro)
## eventLogger
The `eventLogger` uses `Fw::LogPacket` to create its `ComBuffer`, so it looks like
```
type (FW_PACKET_LOG) | time | log buffer
```

# Packet specific
Now, we want to understand the structure of the telemetry buffer or log buffer.


# Tracing the configuration and calls between the components
Framing is called from the following places
* `comIn_handler`: https://github.com/nasa/fprime/blob/devel/Svc/Framer/Framer.cpp#L48
* `bufferIn_handler`: https://github.com/nasa/fprime/blob/devel/Svc/Framer/Framer.cpp#L61

Framing is also reimplemented in `GroundInterfaceComponentImpl::frame_send`: https://github.com/nasa/fprime/blob/devel/Svc/GroundInterface/GroundInterface.cpp#L97
except this does not include the Checksum

# Connecting to the framer
The framer has been invoked by data being passed to the [Svc::Framer](https://github.com/nasa/fprime/blob/devel/Svc/Framer/docs/sdd.md)

This is done through the component configuration.  For example, in the 
[Ref application](https://github.com/nasa/fprime/blob/master/Ref/Top/RefTopologyAppAi.xml#L56) 
you can see a component named `downlink` is defined that is a `Framer`.
```xml
<instance namespace="Svc" name="downlink" type="Framer" base_id="0x4100" base_id_window="0"/>
```

Then, the components that will connect to the `downlink` are also defined
```xml
<instance namespace="Svc" name="eventLogger" type="ActiveLogger" base_id="0xB00" base_id_window="5"/>
<instance namespace="Svc" name="chanTlm" type="TlmChan" base_id="0xC00" base_id_window="0"/>
<instance namespace="Svc" name="fileDownlink" type="FileDownlink" base_id="0x700" base_id_window="9"/>
```

Now that the components are defined, they need actually be connected to the ports of the `downlinker`.
First, the `chanTlm` and `eventLogger` are connected to `comIn`
```xml
<connection name="[unused]">
    <source component="chanTlm" port="PktSend" type="[unused]" num="0"/>
    <target component="downlink" port="comIn" type="[unused]" num="0"/>
</connection>
<connection name="[unused]">
    <source component="eventLogger" port="PktSend" type="[unused]" num="0"/>
    <target component="downlink" port="comIn" type="[unused]" num="0"/>
</connection>
```
And then `fileDownlink` is connected to `bufferSendOut`
```xml
<connection name="[unused]">
<source component="fileDownlink" port="bufferSendOut" type="[unused]" num="0"/>
<target component="downlink" port="bufferIn" type="[unused]" num="0"/>
</connection>
```

# Inner serialized data
So, we now need to examine the implementation of these three objects (`chanTlm`, `eventLogger`, and `fileDownlink`) 
to understand how they create the `const U8* const data` which was passed to the framer.

## chanTlm (TlmChan)
`chanTlm` is of `type=TlmChan`.  We want to look at its `PktSend` port, since that's connected to the downlinker.

We need to look at `TlmChanImplTask.cpp` to see how it serializes its data into the `comBuffer`
It uses a `TlmPacket` (`Fw::TlmPacket m_tlmPacket;` is defined in `TlmChanImpl.hpp`)
```c++
this->m_tlmPacket.setId(p_entry->id);
this->m_tlmPacket.setTimeTag(p_entry->lastUpdate);
this->m_tlmPacket.setTlmBuffer(p_entry->buffer);
this->m_comBuffer.resetSer();
Fw::SerializeStatus stat = this->m_tlmPacket.serialize(this->m_comBuffer);
FW_ASSERT(Fw::FW_SERIALIZE_OK == stat,static_cast<NATIVE_INT_TYPE>(stat));
p_entry->updated = false;
this->PktSend_out(0,this->m_comBuffer,0);
```

We can then check the implementation inside of `TlmPacket.cpp` (with some of the error checking removed)
```c++
#if !FW_AMPCS_COMPATIBLE
        stat = serializeBase(buffer);
#endif
        stat = buffer.serialize(this->m_id);

#if !FW_AMPCS_COMPATIBLE
        stat = buffer.serialize(this->m_timeTag);
#endif

        return buffer.serialize(this->m_tlmBuffer.getBuffAddr(),m_tlmBuffer.getBuffLength(),true);
```

We can see that it uses the macro `FW_AMPCS_COMPATIBLE`.  We can
read [configuring-fprime.md](../UsersGuide/dev/configuring-fprime.md) to understand its usage.  It is disabled
by default, so all of those pieces will be serialized.

The `serializeBase` method comes from `ComPacket` (look at `TlmPacket.hpp` to see that `TlmPacket` extends `ComPacket`)
`TlmPacket` sets its type to `FW_PACKET_TELEM`, so our resulting structure will look like
```
FW_PACKET_TELEM | id | time | buffer
```
where the `id`, `time`, and `buffer` were all passed in to the `TlmChan`, either through the `Recv` or `Get` ports.

Their types are
```c++
FwChanIdType id, Fw::Time &timeTag, Fw::TlmBuffer &val
```

In `FpConfig.hpp`
```c++
typedef U32 FwChanIdType;
```

## eventLogger (ActiveLogger)
`eventLogger` is of `type=ActiveLogger`.   We want to look at its `PktSend` port, since that is connected to the `comIn`
port of the downlinker.

It is insightful to first read the [documentation](../../Svc/ActiveLogger/docs/sdd.md).
We see that it receives the events over the `LogRecv` port.

Then, we can look at the `LogRecv_handler` method within [ActiveLoggerImpl.hpp](../../Svc/ActiveLogger/ActiveLoggerImpl.hpp)
to see how it handles messages.

This just queues the message up by calling
```c++
this->loqQueue_internalInterfaceInvoke(id,timeTag,severity,args);
```

When we look at the implementation of that method (`ActiveLoggerImpl::loqQueue_internalInterfaceHandler`), we see it 
uses a `LogPacket` serializes the inputs (id, timeTag, and buffer) into an output `comBuffer`, which it sends to `PktSend`.
```c++
this->m_logPacket.setId(id);
this->m_logPacket.setTimeTag(timeTag);
this->m_logPacket.setLogBuffer(args);
this->m_comBuffer.resetSer();
Fw::SerializeStatus stat = this->m_logPacket.serialize(this->m_comBuffer);
FW_ASSERT(Fw::FW_SERIALIZE_OK == stat,static_cast<NATIVE_INT_TYPE>(stat));

if (this->isConnected_PktSend_OutputPort(0)) {
    this->PktSend_out(0, this->m_comBuffer,0);
}
```

We can check the `setLogBuffer` and `serialize` methods of `LogPacket`
```c++
void LogPacket::setLogBuffer(const LogBuffer& buffer) {
        this->m_logBuffer = buffer;
    }
```

```c++
SerializeStatus LogPacket::serialize(SerializeBufferBase& buffer) const {

        SerializeStatus stat = ComPacket::serializeBase(buffer);
        if (stat != FW_SERIALIZE_OK) {
            return stat;
        }

        stat = buffer.serialize(this->m_id);
        if (stat != FW_SERIALIZE_OK) {
            return stat;
        }

        stat = buffer.serialize(this->m_timeTag);
        if (stat != FW_SERIALIZE_OK) {
            return stat;
        }

        // We want to add data but not size for the ground software
        return buffer.serialize(this->m_logBuffer.getBuffAddr(),m_logBuffer.getBuffLength(),true);

    }
```

We need to check `ComPacket::serializeBase`
```c++
SerializeStatus ComPacket::serializeBase(SerializeBufferBase& buffer) const {
        return buffer.serialize(static_cast<FwPacketDescriptorType>(this->m_type));
    }
```

For this, we need to understand `FwPacketDescriptorType`.  In [configuring-types](../UsersGuide/dev/configuring-fprime.md), we
can see that the size of this type is up to the user, but by default is configured in
[FpConfig.hpp](../../config/FpConfig.hpp)
```c++
typedef U32 FwPacketDescriptorType;
```

We see that `LogPacket` is a type of `ComPacket`, and it sets its `m_type` field to `FW_PACKET_LOG`
```c++
LogPacket::LogPacket() : m_id(0) {
    this->m_type = FW_PACKET_LOG;
}
```

Finally, we should check `Serializable.cpp` to see how it will serialize `m_id`, `m_timeTag`, and the buffer.

Checking `LogPacket.hpp`, we can find the types of those fields
```c++
protected:
            FwEventIdType m_id; // !< Channel id
            Fw::Time m_timeTag; // !< time tag
            LogBuffer m_logBuffer; // !< serialized argument data
```

We can find the definition of `FwEventIdType` in `FpConfig.hpp`
```c++
typedef U32 FwEventIdType;
```


We can see that for the primitive type `m_id`, `Serializable` just writes the value to the buffer.
```c++

```

For `m_timeTag`, `Fw::Time` implements its own `serialize` method, so we check `Time.cpp`.  We can
see that it uses the macros `FW_USE_TIME_BASE` and `FW_USE_TIME_CONTEXT`.  We can
read [configuring-fprime.md](../UsersGuide/dev/configuring-fprime.md) to understand their usage.

The result (in the default case) is that the `m_timeTag` is serialized as
```
m_timeBase (FwTimeBaseStoreType = U16) | m_timeContext (FwTimeContextStoreType = U8) | m_seconds (U32) | m_useconds (U32)
```
Don't forget to check FpConfig.hpp to confirm the sizes for these types.

For `m_logBuffer`, we check how `Serializable` handles buffers, and see there is an optional parameter that controls whether it first writes the length,
before copying the buffer in.  
```c++
SerializeBufferBase::serialize(const U8* buff, NATIVE_UINT_TYPE length, bool noLength)
```
For the `LogPacket`, we set `noLength=true`, so we just write the buffer without length.


So, to summarize all of that, the serialized form of the log should look like
```
| FW_PACKET_LOG (32 bits) | log packet id | log packet time tag | log packet log buffer
```

### log buffer
We glossed over the log buffer that was passed in to the `ActiveLogger`.  We need to see who is connected
to the `LogRecv` port, since they send it buffers.

We can check `RefTopologyAppAi.xml`, and see 21 different connections to the logger (look for `<target component="eventLogger" port="LogRecv"`)

As an example, lets look at the `Log` port of component `pingRcvr`, which is of type `PingReceiver`.  The
implementation is found in `PingReceiverComponentImpl`.

## fileDownlink (FileDownlink)


# List of packet structures
| Packet        | Generated by   | inner packet structure                                                                                      |
|---------------|----------------|-------------------------------------------------------------------------------------------------------------|
| file          |                |                                                                                                             |
| telemetry     |                |                                                                                                             | 
| FW_PACKET_LOG |                |                                                                                                             |
|               | FprimeProtocol | `FPRIME_START_WORD _ length _ FW_PACKET_FILE _ generic buffer _ checksum`                                   |
|               | LogPacket      | `FPRIME_START_WORD _ length _ FW_PACKET_LOG _ time _ log buffer   _ checksum`                               |
|               | TlmPacket      | `FPRIME_START_WORD _ length _ FW_PACKET_TELEM _  telemetry channel id _ time _ telemetry buffer _ checksum` |
|               |                |                                                                                                             |


# Open questions
* In FprimeFraming::frame, the last line is `m_interface->send(buffer);`.  Does this modify the outgoing
data in any way, for example adding the length?
* Why does `GroundInterfaceComponentImpl::frame_send` re-implement framing, instead of using the FPrimeFraming?
* Best way to include source code links in this documentation
* How did I get to `Serializable.cpp` from the method call `buffer.serialize(`
* How to find usages of the `logOut` port within the `SignalGen` components like `SG1`?
* Could we standardize `ComBuffer`, instead of its format being up to the senders?