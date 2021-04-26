---
title: "Wireguard Install on Edgerouter-X"
date: 2021-04-24T08:33:45-04:00
---

I used to have OpenVPN conigured for my home network. This allowed access to machines at my house without having to expose my entire home network to the internet. It comes in super-handy if you forgot something on a machine at home, or need to remote desktop into your partner's computer. Unfortunatley, the certificates I generated during setup only had a few years lifetime, and expired about or year or two ago. Trying to understand how to maintain OpenVPN and issue new certs ended up being too much hassle, so I abadoned it.

Enter [WireGuard](https://wireguard.com). I'd heard about WireGuard for a long time, with the most recent(-ish?) news that it was integrated into the Linux kernel. I kept hearing that it was very easy to configure and maintain. I had OpenVPN running on my EdgeRouter-X (ER-X). It was really nice to have it running on my router, because it's always on and connected directly to my modem. If I were going to run WireGuard, I'd want to do it the same way.

Sure enough, someone created a [WireGuard package](https://github.com/WireGuard/wireguard-vyatta-ubnt) for the ER-X, and it even looks like WireGuard themselves have picked-up maintenance of the project. I didn't even know you could install packages on the EdgeRouter-X, so this was a whole new world to me.

So, I blindly followed the directions on the GitHub page and installed WireGuard package, mostly following the configuration specified on the page.

First, I fetched the package with `wget` and installed via `dpkg` (who knew the ER-X had `dpkg`!?):

```bash
curl -OL https://github.com/WireGuard/wireguard-vyatta-ubnt/releases/download/${RELEASE}/${BOARD}-${RELEASE}.deb

sudo dpkg -i ${BOARD}-${RELEASE}.deb
```

One thing to note, here. My EdgeRouter was running a v1 firmware. When I downloaded the binary, it informed me that I needed to upgrade the firmware on my ER-X. What I didn't realize is the Github page provides binaries for *BOTH* ER-X v1 and v2 firmwares (I had just selected the v2). So, I didn't have to upgrade the firmware after all. But I guess it never hurts.

This is the part where I should have read up on WireGuard and not just copy-pasted commands from the GitHub page. It wasn't catastrophic, I just really didn't understand what I was doing. For the sake of understanding, let's break down what's happening on that page a bit (copied below):

```bash
wg genkey | tee /config/auth/wg.key | wg pubkey >  wg.public

configure

set interfaces wireguard wg0 address 192.168.33.1/24
set interfaces wireguard wg0 listen-port 51820
set interfaces wireguard wg0 route-allowed-ips true

set interfaces wireguard wg0 peer GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc= endpoint example1.org:29922
set interfaces wireguard wg0 peer GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc= allowed-ips 192.168.33.101/32

set interfaces wireguard wg0 peer aBaxDzgsyDk58eax6lt3CLedDt6SlVHnDxLG2K5UdV4= endpoint example2.net:51820
set interfaces wireguard wg0 peer aBaxDzgsyDk58eax6lt3CLedDt6SlVHnDxLG2K5UdV4= allowed-ips 192.168.33.102/32
set interfaces wireguard wg0 peer aBaxDzgsyDk58eax6lt3CLedDt6SlVHnDxLG2K5UdV4= allowed-ips 192.168.33.103/32

set interfaces wireguard wg0 private-key /config/auth/wg.key

set firewall name WAN_LOCAL rule 20 action accept
set firewall name WAN_LOCAL rule 20 protocol udp
set firewall name WAN_LOCAL rule 20 description 'WireGuard'
set firewall name WAN_LOCAL rule 20 destination port 51820

commit
save
exit
```

Taking it piece-by-piece:

```bash
wg genkey | tee /config/auth/wg.key | wg pubkey >  wg.public
```

This generates the servers public and private keys. The private key saved in `/config/auth/wg.key` and the public key is derived from the private key and stored as `wg.public` in the current directory. That's it! No certificate authority to configure, no serverside certificates to generate and maintain, no OpenSSL CLI syntax to remember. This command alone generates the server's key-pair. This is at least 1000 times better than OpenVPN's complex certificiate-based authentication setup.

```bash
configure
```

This enters configuration mode on the ER-X.

```bash
set interfaces wireguard wg0 address 192.168.33.1/24
```

Okay, so this is where I originally got confused. This part of the config sets the VPN address for the server, and the netmask. This doesn't need to match your subnet for your existing wired network (and probably shouldn't). `192.168.33.1/24` sets the WireGuard server IP address to `192.168.33.1` and treats all `192.168.33.X` addresses as valid VPN addresses. In my case, I changed this to `10.0.0.1/24` (server address of `10.0.0.1` with valid VPN addresses of `10.0.0.X`).

```bash
set interfaces wireguard wg0 listen-port 51820
set interfaces wireguard wg0 route-allowed-ips true
```

Just taking these two at the same time. These lines conigure the WireGuard server to listen on port `51820` and will automatically route allowed IPs (covered below, when peers are described). I'm not 100% sure about this, but if you set `route-allowed-ips` to `false`, you'll explicitly need to add routes for your VPN connections. I left this as `true`.

```bash
set interfaces wireguard wg0 peer GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc= endpoint example1.org:29922
set interfaces wireguard wg0 peer GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc= allowed-ips 192.168.33.101/32
```

These lines configure a peer (a.k.a. client). `GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc=` is the public-key of the client. `example1.org:29922` is the remote IP address the client is connecting from. `allowed-ips 192.168.33.101/32` configures the server to only accept the client identified by the public key to use the IP address `192.168.33.101`.

One note, here. You need not configure the `endpoint`. If you're using WireGuard to connect remotely to your home network while on-the-go, you shouldn't configure the endpoint at all (I didn't).

I'm going to skip the other peer configs as they're just additional examples of the same configuration.


```bash
set interfaces wireguard wg0 private-key /config/auth/wg.key
```

Pretty straightforward; this just sets the private key for the server (which was generated in the first step above).

```bash
set firewall name WAN_LOCAL rule 20 action accept
set firewall name WAN_LOCAL rule 20 protocol udp
set firewall name WAN_LOCAL rule 20 description 'WireGuard'
set firewall name WAN_LOCAL rule 20 destination port 51820
```

This adds a firewall rule to the WAN interface to allow incoming UDP traffic on port 51820.

```bash
commit
save
exit
```

Also pretty self-explainatory, this commits the config changes to the ER-X, persists them and exits configuration mode.

That's all there is to configuring the server. The last bit of the puzzle is the client configuration. I won't go into any detail about how to install WireGuard on your computer (or phone or tablet); that's covered pretty well elsewhere.

```bash
[Interface]
PrivateKey = A_PRIVATE_KEY
Address = 192.168.33.101/32

[Peer]
PublicKey = SERVER_PUBLIC_KEY
AllowedIPs = 192.168.33.0/24
Endpoint = SERVER_ENDPOINT:PORT
```

The client config is pretty small. Replace `A_PRIVATE_KEY` with a private key you've generated from the peer config. It should correspond to the public key used in the ER-X peer configuration above (e.g. `GIPWDet2eswjz1JphYFb51sh6I+CwvzOoVyD7z7kZVc=`).

The server public key is the public key we generated in the first step (e.g. the contents of `wg.public`).

The server endpoint is the IP address of DNS address of your ER-X, and the port is the port we configured above for the server (e.g. `51820`).

That's it. If everything worked correctly, you should now have a WireGuard server running on your ER-X with a sample WireGuard client config you should be able to use to connect.
