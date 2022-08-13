---
title: "Go and UDP"
date: 2022-03-21T16:45:13-04:00
draft: true
---

I've got many Rapsberry Pis lying around not doing much. I wanted to devise a project, however simple, where I could write some networking code for the pis. I've been using Go at work for the past two years, but really haven't dug into much of the lower-level portions of the language. When I work in large codebases I tend to find myself working primarily with existing business objects and libraries. This doesn't allow for much low-level tinkering.

While trying to keep things as simple as possible, I decided I wanted to write what I've called "Go UDP Ping Pong." Assuming we have two clients that are pre-configured with the other's IP address, the idea is:

* open a UDP socket for incoming connections,
* if a packet isn't received after some random number of seconds, launch a packet at the other client,
* once a packet has been received on the listening port send it back to the other client.

So, it's kind of like ping pong where you can't see past the middle of the table. Also, you don't know if the other player has served or not, so you randomly have to serve up a ball. But once a ball comes across the table you smack it back until (hopefully) it comes back again. At best, a poor analogy.

So, how do you open a UDP port in Go? Turns out there are a couple ways. You can open a UDP port, or you can listen on a UDP port. Let's look at opening a port:

```go
conn, err := net.DialUDP("udp", nil, udpAddr)
if err != nil {
	panic(err)
}
```

`net.DialUDP` returns a [`UDPConn`](https://pkg.go.dev/net#UDPConn). `UDPConn` implements both the [`Conn`](https://pkg.go.dev/net#Conn) and [`PacketConn`](https://pkg.go.dev/net#PacketConn) interfaces.

And now for listening on a UDP port:
```go
pc, err := net.ListenPacket("udp", lH)
if err != nil {
	log.Fatal(err)
}
```

`net.ListenPacket` returns a [`PacketConn`](https://pkg.go.dev/net#PacketConn).

The first part I wanted to approach was sending the initial packet, so I used `net.DialUDP`. However, I quickly ran into an issue. Take a look:

