---
layout: post
title: "Build a Network Stack in Go - Part 3: Buffers and Headers"
date: 2022-08-21
categories: networking
tags: networking
usemathjax: false
---

This is the third in a series of blog posts detailing how to write a network stack in Go. This post will cover the core data structure used to pass network data up and down the stack, as well as network protocol headers, which carry metadata used by the network protocols.

## Socket Buffers

A socket buffer is a data structure that gets passed between the network stack layers and protocols. It contains the raw network packet plus some extra data used by the network stack.

If we consider some data being sent from userspace to another host over the network, that data will start out as raw data, then as it's processed by each protocol along the stack, will have successive network protocol headers prepended to it (encapsulation).

If you recall from [post 2](/networking/2022/08/14/networking-stack-part-2.html), our layer and protocol implementations looked something like this:

```go
type Layer struct {
    Protocols map[ProtocolType]Protocol
    RxChan chan *SkBuff
    TxChan chan *SkBuff
}

type MyProtocol struct {
    RxChan chan *SkBuff
    TxChan chan *SkBuff
}

// Implement Protocol interface...
```

As you can see, we have two channels that shuttle around data our data, which is of type `*SkBuff` (we use a pointer type because our data structure will be modified by each protocol). 

As for the buffer itself, we know it definitely needs a buffer to hold the raw bytes:

```go
// SkBuff v1
type SkBuff struct {
    Data []byte
}
```

In addition to the raw bytes it is also useful to have access the the parsed protocol headers. Let's take a detour to discuss those right now.

## Protocol Headers

Each protocol in the network stack has an associated header. As the data is passed up and down the stack, these headers get stripped or appended, respectively. In Go it looks like this:

```go
type IPv4Header struct {
	Version        uint8
	IHL            uint8
	TypeOfService  uint8
	TotalLength    uint16
	Identification uint16
	Flags          uint8
	FragmentOffset uint16
	TTL            uint8
	Protocol       uint8
	HeaderChecksum uint16
	SourceIP       net.IP
	DestinationIP  net.IP
}

func (h *IPv4Header) Unmarshal(data []byte) error {
    // parse bytes into IPv4 header struct
}

func (ip *IPv4) HandleRx(skb *SkBuff) {
    // At this point skb.Data has been stripped of the link layer header,
    // but the network layer header still remains, so we need to strip the 
    // bytes and unmarshal into our IPv4Header struct

    h := &IPv4Header{}
    if err := h.Unmarshal(skb.Data); err != nil {
        // handle unmarshal error
    }

    skb.StripBytes(h.IHL * 4)
    
    // Perform rest of IPv4 handling logic...
}
```

Here we are using the IPv4 protocol as an example but the concept is the same for every other protocol. 

Now as the packet is moving through the stack, it may be necessary for a lower level protocol to know details about a higher level protocol, or vica versa. For example, the TCP protocol maintains connections based on the 4-tuple `(local ip, local port, remote ip, remote port)`, so the TCP protocol needs access to the IP header to do it's business. Thus, when we unmarshal a header we will also stash it in the socket buffer for use later down the line. 

Now, because we want our network stack to be extensible and support many protocols, we'll don't stash the specific protocol header struct, but rather as a common header interface.

```go
type Header interface {
	Marshal() []byte
	Unmarshal([]byte) error
	GetType() ProtocolType
}

type L2Header interface {
	Header
	GetSrcMAC() net.HardwareAddr
	GetDstMAC() net.HardwareAddr
	GetL3Type() ProtocolType
}

type L3Header interface {
	Header
	GetSrcIP() net.IP
	GetDstIP() net.IP
	GetL4Type() ProtocolType
}

type L4Header interface {
	Header
	GetSrcPort() uint16
	GetDstPort() uint16
}
```

With this in place, we're ready to complete the implementation of the `SkBuff`:

```go
// SkBuff v2
type SkBuff struct {
	Data         []byte
	ProtocolType ProtocolType
	L2Header     L2Header
	L3Header     L3Header
	L4Header     L4Header
}
```

All the code from this post is available in the github repository: [https://github.com/mattcarp12/matnet](https://github.com/mattcarp12/matnet).

## Socket Buffers in other networking stacks

The socket buffer data structure is present in other networking stacks:
- Linux kernel uses a `struct sk_buff` to hold packet metadata. This book is a great reference on [linux network internals](https://www.amazon.com/Understanding-Linux-Network-Internals-Networking-ebook/dp/B0043EWV3S).
- BSD uses `struct mbuf`. The wonderful [TCP/IP Illustrated, Vol. 2](https://www.amazon.com/TCP-IP-Illustrated-Implementation-Vol/dp/020163354X) gives all the glorious details.