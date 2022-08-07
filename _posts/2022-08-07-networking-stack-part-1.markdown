---
layout: post
title: "Build a Network Stack in Go - Part 1: Concepts"
date: 2022-08-07
categories: networking
tags: networking
usemathjax: false
---

This is the first in a series of blog post detailing how to write a network stack in Go.
This post introduces the basic concepts of a network stack and sets the stage for the rest of the series.

## What is a network stack?

Simply stated, a network stack is a collection of networking protocols and the code to orchestrate their interaction.

Surely you've seen this picture (or something similar) before:
![netstack](/assets/Network-Stack.jpg)

When taking about a network stack, we make the distinction between the part handled in the kernel and the part handled in the user space.
Usually the kernel handles all the lower layers, starting with the network device driver, and ending with the transport protocol (TCP, UDP, etc).

Above the kernel, the user space handles the higher layers, including the application layer (HTTP, SMTP, etc).

Between them is the **Network API**, which is a set of functions that allow the user space to interact with the kernel. The best known example of this is the [Berkely Sockets API](https://en.wikipedia.org/wiki/Berkeley_sockets) (aka POSIX sockets). This is a set of system calls provided by the kernel that allow the user space to interact with the network stack.

We'll be building our own network stack, so we'll also need to implement our own Network API!

## Layers and Protocols

There are two main models of a network stack we'll consider: the [OSI model](https://en.wikipedia.org/wiki/OSI_model) and the [Internet protocol suite](https://en.wikipedia.org/wiki/Internet_protocol_suite) (aka TCP/IP protocol suite).
The OSI model has 7 layers and the Internet protocol suite has 4 layers. The OSI model is more general, applying to more than just the TCP/IP.

For this project I've ended up creating a blend of the two: I attemped to keep the generality of the OSI model, but also keeping the 4 basic layers of the TCP/IP protocol suite.

While there will be more detail on this in later posts, I want to set the definitions at the outset.

A **layer** is a collection of protocols. Thats it. Just a dumb container.

A **protocol** is a single piece of functionality that receives packets, performs some processing, and sends packets. Protocols send packets up or down the stack, depending on whether a packet is received from the network or sent by the application.

## How to build a network stack

For this project we won't be building an entire kernel, or even writing any kernel code. So how do we build a network stack? We'll be building a **Userspace Network Stack**. This was a very intriguing concept for me, and initially took a while for it to click, but it's not that hard.

To build a userspace network stack, all we need to do is make sure the packets that come in from "the wire" don't go to the kernel network stack, but instead get diverted to our own wonderful network stack. To do this we'll leverage a **virtual network interface**. In linux, this is called a [tap device](https://en.wikipedia.org/wiki/TUN/TAP). More details will come in a later post in this series.

## Resources

This [series of blog posts](https://www.saminiir.com/lets-code-tcp-ip-stack-1-ethernet-arp/) was the original inspiration for this project, and I borrowed heavily from it. It's a great introduction to building a network stack.

This [paper](https://arxiv.org/abs/1603.05636) describes the author's attempt at building a network stack in Go. It's a great paper, and I highly recommend it.

Here is the [source code](https://github.com/mattcarp12/matnet) for this project.
