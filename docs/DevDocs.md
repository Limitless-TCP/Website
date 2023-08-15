# Developer Documentation for Contributors
This documentation is made to help potential contributors navigate the structure of Limitless-TCP.

This is based off of how the JavaScript version works, other versions will mostly be built off of this,
with a few optimization changes.

## Table of Context
| Section          | Description                                                                                              | Link                          |
|------------------|----------------------------------------------------------------------------------------------------------|-------------------------------|
| System Overview  | This section provides an overview of the entire library with a breakdown of its systems                  | <a href="#section-1">View</a> |
| Packet Structure | This section provides the structure for each of the packets that are sent between the server and clients | <a href="#section-2">View</a> |

<h1 id="section-1">System Overview</h1>
This section overviews each of the systems in Limitless-TCP and describes how they work.

## System Startup

### TCPServer
The TCPServer class will only listen if the port is valid and the listen() method is called.

As for the settings object, if any field is left blank then the settings are automatically set to false,
and if the object itself is not present, all values are defaulted to false.


### TCPClient
When the TCP Client connects to the server, it waits for a packet from the server before emitting the
connection event, this packet contains the settings for the network, these are the settings that are
set by the server.

If this packet is never received, that means the server is not a Limitless-TCP Server, in that case,
the client functions as a normal tcp client.

## Heartbeats
Heartbeats are listened for by a separate listener that is called when the server or client listens or connects.

### TCPServer
Every second, the server sends a heartbeat packet to each client that is connected. If the client is online,
the packet will be reflected back to the server, the server then marks the packet as received. 900 milliseconds after the heartbeat is sent,
the server checks if the packet was received, if it was then all counters for that client are reset.

#### Counters
Each client has a heartbeat counter, this counter is incremented every time a heartbeat is marked as not received. This happens 8 times before
the client is marked as disconnected and a heartbeat error is thrown. This is made like this to account for any errors that can occur causing 
the packet to be dropped.

### TCPClient
When a client receives a heartbeat and returns a heartbeat packet, a lastHeartbeat variable is set to the current time (in unix).
(This value is set to the startup time in unix by default)

Every second, the client checks to see if the lastHeartbeat value was set to more than 8 seconds ago.
If it is set to more than 8 seconds ago then the server is marked as disconnected and a heartbeat error is thrown.

## Anti-Packet Stacking
In order to prevent packets from stacking, we add a identifier after each packet in order to split up stacked packets, and since stacked packets always arrive as one string,
we can split up the packets using a .split() function, then parse the JSON objects.

### Packet Splitter Identifier
```
<PacketSplitter>
```

**Stacked packets would be received as:**
```
{ "type": "tcpsjs-heartbeat" }<PacketSplitter>{ "type": "tcpsjs-heartbeat" }<PacketSplitter>
```

## Anti-Packet Halving
In order to prevent packets from being received halved, we have to rebuilt to packets as they come in, the way we can tell if a packet is halved is if we try to parse a json
and it fails, the packet is invalid, in this case we can add it to a string of halved packets and attempt to split and parse it again. This keeps going until every packet has been parsed.

There is a garbage collector that purges inactive packet strings, so if a client or the server disconnects, the string of packets can be emptied so it doesnt take up space.

### TCPServer
On the server, there is an array of clients that is used to store transactions and halved packets for each client separately

### TCPClient
On a client there is an array for transactions and a string for halved packets that is accessed globally by the packet parser.

## Packet Sending and Chunking

### Sending a normal packet
When sending a normal packet, all you need to do is send the <a href="#messagePacket">Message Packet</a> with a packet splitter after it.

### Sending a compressed packet
When sending a compressed packet, it is basically the same as sending a normal packet, the difference is that the data field
is a Buffered ZLib compressed string. Any ZLib library will work to compress packets

### Chunking and sending a packet

**Normal:**
Chunking occurs when a packet is more than 14000 bytes in size, when this happens, the packet is split into packets that are split into chunks of 14000 bytes.
These packets are then sent every millisecond until finished using the <a href="#chunkPacket">Chunk Packet</a>.
For the last chunk, the "last" key of the chunk packet must be set to true so the recipient knows when to assemble the packet.

**Compressed:**
Sending a compressed packet is the same as sending a normal packet, the difference is that the 14000 bytes is checked after the packet is compressed and the
compressed buffer is what is chunked.

## Receiving a packet
When a packet is received, it is immediately send to a worker thread, so it can be parsed in the background ensuring that the heartbeats and the users project run as maximum
efficiency.

### Receiving a normal packet
When a packet is sent to a worker thread, it goes through the process of being split or built if it was halved. If the parsed packet is not a chunk or message, it is discarded
because those are handled in a separate listener. If the packet is a normal message, it attempts to parse it as a json, then it emits the data event that is then sent to the user.

### Receiving a compressed packet
A compressed packet works the same as a normal packet, the only difference being that you have to decompress it before emitting the event.

### Receiving a chunk of a packet
When a chunk of a packet is received, it is added to a chunks array which is a part of a transaction object. There is a transaction object for each transaction with
a unique ID. This allows for multiple transactions to occur at once. When a chunk is received with the "last" key set to true, the worker thread will then put all the chunks
together, attempt to parse it as a json and then emits the data event to the user.

In the case of it being compressed, you would concat the buffers to one buffer, then decompress it before emitting the data event to the user.

## Connected Sockets and Total Sockets Arrays
Each server instance of limitless TCP has an array containing each client that is connected to the server and every
client that is and has been connected to the server since the last time it was started.

This is mainly because I got tried of making them myself, clients are removed from the connected clients array when they
disconnect or time out due to heartbeats.

<h1 id="section-2">Packet Structure</h1>
All packets sent between the server and the client use JSON formatting because the JSON format is universal

Each packet here is followed by a `<PacketSplitter>` object

## Connection Packet
This packet is sent from the server to the client on connection. 
If this packet is never received then the client acts as a normal tcp client
```json
{ 
  "type": "tcpsjs-connect",
  "data": { 
    "useHeartbeat": bool, 
    "useCompression": bool, 
    "useChunking": bool
  }
}
```

## Heartbeat Packet
This is the packet that is sent from the server to the client and then bounced back to the server to determine heartbeat timeouts
```json
{ 
  "type": "tcpsjs-heartbeat"
}
```

<h2 id="messagePacket">Message Packet</h2>
This is the packet that is sent to a client or server and then called as an event
```json
{ 
  "type": "tcpsjs-packet", 
  "data": "" //This is either a zlib compressed buffer or a string
}
```

<h2 id="chunkPacket">Chunk Packet</h2>
```json
{ 
  "type": "tcpsjs-chunk", 
  "transactionId": uuid, //This is so multiple chunkings can be done at one time 
  "chunkNum": number, //This is the current chunk number (starting from 0)
  "chunkAmount": number, //This is how many chunks there are in total 
  "last": bool, //This is how the recipient can tell when to assemble the packet 
  "data": Buffer //(This is the actual chunk data)
}
```