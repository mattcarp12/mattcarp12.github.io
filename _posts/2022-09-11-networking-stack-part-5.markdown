---
layout: post
title: "Build a Network Stack in Go - Part 5: Socket API"
date: 2022-09-04
categories: networking
tags: networking
usemathjax: false
---

This article is the last in the series "Build a Network Stack in Go". This article will introduce the Socket API.

The Socket API is how the enduser interacts with the lower levels of the network stack. For this project I have used the Berkeley Socket API as inspiration for my own Socket API, but written in Go.

Essentially our Socket API implementation will be a Go package that a user can import into their source code to write programs that interact with our Network Stack. Here is a sneak peek:

```go
package main

import s "github.com/mattcarp12/matnet/api"

func main() {
    s.Socket(s.SocketTypeStream) // Make a new socket in our network stack

    // Perform network communication
}
```

Since this Network Stack is run in userspace, we need a way to do Inter Process Communication (IPC).

Here is how traditional socket api works:

![trad_sock](/assets/trad_socket_api.jpg){: width="600" }

(1) is the the system call made by your program running in userspace, that is handled by the operating system kernel. 

Here is how our socket api works:

![user_sock](/assets/user_socket_api.jpg){: width="600" }

Here, instead of (1) being a system call directly into the kernel, we call into the userspace network stack, which then in turn calls into the operating system in (2). Our goal in developing the userspace network stack is for (2) to be hidden (transparent) to the user.

In actuality, we are calling into the kernel to do IPC, so the userspace network stack actually looks something like this:

![user_sock_actual](/assets/user_socket_api_actual.jpg){: width="600" }

Here, (1) and (2) represent the system call used to do IPC, and (3) is the actual system call made by the userspace network stack. Cool! 


## Implementing the Socket API

To implement the Socket API, we need to implement a server that listens for requests from userspace programs. The widely used method to do this is with [Unix Domain Sockets](https://en.wikipedia.org/wiki/Unix_domain_socket). Here is a preview of how this is done in golang:

```golang
const ipcAddr = "/tmp/gonet.sock"

// serve ...
func (ipc *IPC) serve() {
    // Start the Unix Domain Socket server
	listener, err := net.Listen("unix", ipcAddr)
	if err != nil {
		ipcLog.Fatal(err)
	}

	// change file permission so non-root users can access
	sockPermission := 0o777
	if err := os.Chmod(ipcAddr, os.FileMode(sockPermission)); err != nil {
		ipcLog.Fatal(err)
	}

	defer listener.Close()

    // Accept loop...
	for {
		conn, err := listener.Accept()
		if err != nil {
			continue
		}

		// Start goroutine to handle connection
		go conn.handleConnection()
	}
}

// handleConnection ...
// this is the goroutine that will handle the connection
// to the client process. It listens for Syscall Requests
// from the client and dispatches them to the socket layer.
func (iconn *ipcConn) handleConnection() {
	reader := bufio.NewReader(iconn.conn)

	for {
		// Read request
		var req SockSyscallRequest

		err := req.Read(reader)
		if err != nil {
			iconn.close()

			return
		}

		// Set the connection ID
		req.ConnID = iconn.id

		// Send request to socket layer
		iconn.socketLayer.SyscallReqChan <- req

		// Wait for response
		resp := iconn.getResponse()

		// Write response
		rawResp := append(resp.Bytes(), '\n')
		if _, err = iconn.conn.Write(rawResp); err != nil {
			ipcLog.Printf("Error writing response: %s", err)
		}
	}
}
```

As you can see, we're using basic socket programming to implement our Socket API!

So all our Socket API is is really just a library for making requests to this server and reading responses.

I hope you've found this series on userspace networking informative. This project was very enjoyable and enlightening for me and I suggest all backend software engineers to dig deep on socket programming!