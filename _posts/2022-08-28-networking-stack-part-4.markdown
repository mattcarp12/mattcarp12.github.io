---
layout: post
title: "Build a Network Stack in Go - Part 4: Userspace Networking"
date: 2022-08-28
categories: networking
tags: networking
usemathjax: false
---

In this post we'll discuss a very important topic: how packets get from the outside network into our network stack.

## Traditional Networking Stacks

As we already know, traditionally the network stack is located in the kernel. That is to say, when a packet comes into our host from the outside network, it is received and processed by the kernel, then passed to userspace. More specifically, the kernel is in charge of layers 2 through 4, typically meaning Ethernet, IP, and TCP protocols.

Here is a diagram of the usual situation:
![netstack](/assets/trad_net_stack.jpg){: width="600" }

## Userspace Network Stack

With userspace networking (how our network stack is built), each packet is sent directly to userspace. Well, not exactly. The real story is more complicated than that. 

What happens is we make a **virtual network interface** in the kernel. This is a special network interface that is tied directly to a single running process. Other than that it works like any other network interface, i.e. it has a MAC address and an IP address. So when packets come in from the network (through an actual physical network interface) that are destined for our virtual network interface, the kernel passes them along and they go straight to our userspace process (i.e. our network stack). This is how we can have a full fledged network stack running in userspace!

Here is a diagram showing how it works:
![netstack](/assets/user_net_stack.jpg){: width="600" }

## Virtual Network Interfaces

Virtual network interfaces are ubiquitous is modern computing. Ever used Docker or VPN software? Then you've used a virtual network interface.

All modern operating systems (Linux, Windows, Mac) support creating virtual network interfaces. For this project I'm only supporting Linux, but please make a PR if you want Windows support ;).

In Linux there are two basic types of virtual interfaces: [TAP and TUN](https://www.kernel.org/doc/html/v5.8/networking/tuntap.html). TAP interfaces are Layer 2 devices, i.e. they have a MAC address and receive ethernet frames. TUN interfaces are Layer 3 devices, i.e. they have an IP address and receive IP packets. For our network stack we're aiming for a deep understanding of the network stack, so we're using a TAP device.

You can create a virtual interface either on the command line, or programmatically with system calls. Since we're using Golang, we'll need to use the `syscall` package in the Go standard library. Here is just a snippet of the code involved:

```go
// Helper function for IOCTL system calls
func ioctl(fd uintptr, request uintptr, argp uintptr) error {
	_, _, errno := syscall.Syscall(syscall.SYS_IOCTL, fd, request, argp)
	if errno != 0 {
		return os.NewSyscallError("ioctl", errno)
	}

	return nil
}

// This function creates the virtual interface using an IOCTL syscall
func createInterface(fd uintptr, ifName string, flags uint16) (string, error) {
	var req ifReq
	req.Flags = flags
	copy(req.Name[:], ifName)

	if err := ioctl(fd, syscall.TUNSETIFF, uintptr(unsafe.Pointer(&req))); err != nil {
		return "", err
	}

	createdIFName := strings.Trim(string(req.Name[:]), "\x00")

	return createdIFName, nil
}

// This function sets the owner and group of the virtual interface,
// along with the persistance setting 
func setDeviceOptions(fd uintptr, config Config) error {
	if config.Permissions != nil {
		if err := ioctl(fd, syscall.TUNSETOWNER, uintptr(config.Permissions.Owner)); err != nil {
			return err
		}

		if err := ioctl(fd, syscall.TUNSETGROUP, uintptr(config.Permissions.Group)); err != nil {
			return err
		}
	}

	// set clear the persist flag
	value := 0
	if config.Persist {
		value = 1
	}

	return ioctl(fd, syscall.TUNSETPERSIST, uintptr(value))
}
```

This article in particular was helpful for my understanding of virtual network interfaces: [https://backreference.org/2010/03/26/tuntap-interface-tutorial/](https://backreference.org/2010/03/26/tuntap-interface-tutorial/).

All the code for this project is located in my [github repo](https://github.com/mattcarp12/matnet).

In the next article we'll wrap up this series with a look at the Socket API. 

Happy coding!