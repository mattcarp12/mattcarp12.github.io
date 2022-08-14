---
layout: post
title: "Build a Network Stack in Go - Part 2: Layers and Protocols"
date: 2022-08-14
categories: networking
tags: networking
usemathjax: false
---

This is the second in a series of blog posts detailing how to write a network stack in Go.

## Network Layers

As stated in part 1, a network stack __layer__ is a collection of protocols. We can model our layer in Go like this:

```go
type ProtocolType int

// Layer v1
type Layer struct {
    Protocols map[ProtocolType]Protocol
}
```

Thats it! Well, not really. We'll need to add a bit of functionality to this struct.

Here is a diagram of the network layer abstraction:

![Network layer](/assets/Network-Layer.jpg)

### Sending and receiving packets

A network layer has two jobs:

- Receive packets from lower layers and dispatch to the appropriate protocol
- Receive packets from higher layers and dispatch to the appropriate protocol

The way we'll send and receive packets is by using goroutines and channels. Each goroutine will read packets from a channel, and dispatch them to the appropriate protocol, based on the protocol type. We'll talk more about protocols later, but just know that protocols have send and receive channels too.

In order to do this we'll need to add a few fields to our `layer` struct. Don't worry about what an `SkBuff` is yet, we'll get to that in a later post.

```go
// Layer v2
type Layer struct {
    Protocols map[ProtocolType]Protocol
    RxChan chan SkBuff
    TxChan chan SkBuff
}
```

Here is the code for sending and receiving packets:

```go
func (layer Layer) GetProtocol(protocolType ProtocolType) (Protocol, error) {
    protocol, ok := layer.protocols[protocolType]
    if !ok {
        return nil, ErrProtocolNotFound
    }

    return protocol, nil
}

func (layer Layer) RxDispatch() {
    for {
        // Layer reads SkBuff from it's RxChan
        skb := <-layer.RxChan()

        // Dispatch skb to appropriate protocol
        protocol, err := layer.GetProtocol(skb.GetType())
        if err != nil {
            continue
        }

        // Send skb to protocol
        protocol.RxChan() <- skb
    }
}

func (layer Layer) TxDispatch() {
    for {
        // Layer reads skb from it's tx_chan
        skb := <-layer.TxChan()

        // Dispatch skb to appropriate protocol
        protocol, err := layer.GetProtocol(skb.GetType())
        if err != nil {
            continue
        }

        // Send skb to protocol
        protocol.TxChan() <- skb
    }
}


func (layer Layer) StartLayer() {
    go layer.RxDispatch()
    go layer.TxDispatch()
}
```

### What about this `ProtocolType`?

In order for the layer to know which protocol to dispatch packets to, we need to know which protocol type the packet is. Put another way, as a packet moves up the stack,  __a lower layer needs to know what higher level protocol to send the packet to__.

This functionality is implemented in most network protocols. Here are a few examples.

- An ethernet frame (layer 2) includes the `EtherType` field, which tells the higher layer (layer 3) what type of packet it is (IP, ARP, etc).
- An IP (layer 3) packet includes the `Protocol` field, which tells the higher layer (layer 4) what type of packet it is (TCP, UDP, etc).

## Network Protocols

A __protocol__ is a single piece of functionality that receives packets, performs some processing, and sends packets. Protocols send packets up or down the stack, depending on whether a packet is received from the network or sent by the application.

Since we're going to be implementing several protocols in our network stack, and want to keep it open and extensible, we'll model a protocol using an interface.

```go
// Protocol v1
type Protocol interface {
    HandleRx(skb *SkBuff)
    HandleTx(skb *SkBuff)
}
```

Now, as I said earlier, each protocol has two channels it reads from: one from the lower layer, and one from the higher layer.

```go
// Protocol v2
type Protocol interface {
    HandleRx(skb *SkBuff)
    HandleTx(skb *SkBuff)
    RxChan() chan *SkBuff
    TxChan() chan *SkBuff
}
```

Now when a protocol processes a packet, it'll need to send the packet to the next layer (either up or down the stack). So we'll need two more methods in the protocol struct.

```go
// Protocol v3
type Protocol interface {
    HandleRx(skb *SkBuff)
    HandleTx(skb *SkBuff)
    RxChan() chan *SkBuff
    TxChan() chan *SkBuff
    RxUp(skb *SkBuff)
    TxDown(skb *SkBuff)
}
```

### Creating a new protocol

So to add a new protocol to our network stack, we'll need to implement the `Protocol` interface. The `HandleRx` and `HandleTx` methods will implement the specific logic for the protocol, but most protocols follow a similar pattern. Here is an example implementation of a protocol's `HandleRx` method:

```go
type MyProtocol struct {}
type MyProtocolHeader struct {}

func (protocol MyProtocol) HandleRx(skb *SkBuff) {
    // Create a new header object and unmarshal the skb into it
    header := MyProtocolHeader{}
    if err := protocol.UnmarshalHeader(skb, &header); err != nil {
        return
    }

    // Strip the header from the skb
    skb.Data = skb.Data[len(header):]

    // Check the header's destination address is correct
    if header.DestinationAddress != protocol.Address {
        return
    }

    // Verify the header's checksum
    if err := protocol.VerifyHeader(skb, &header); err != nil {
        return
    }
    
    // Pass the skb to the higher layer
    protocol.RxUp(skb)
}
```

All the code for this project is located [here](https://github.com/mattcarp12/matnet).

In the next article, we'll talk more about the data buffers that are used to send data up and down the stack.
