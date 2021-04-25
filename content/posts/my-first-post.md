---
title: "Wireguard Install on Edgerouter-X"
date: 2021-04-24T08:33:45-04:00
draft: true
---

I used to have OpenVPN conigured for my home network. This allowed me to machines at my house without having to expose my entire home network to the internet. It comes in super-handy if you forgot something on a machine at home, or need to remote desktop into your partner's computer. Unfortunatley, the certificates I generated during setup only had a few years lifetime, and expired about or year or two ago. Trying to understand how to maintain OpenVPN and issue new certs ended up being too much hassle, and I abadoned it.

Enter [Wireguard](https://wireguard.com). I'd heard about Wireguard for a long time, with the most recent(-ish?) news that it was integrated into the Linux kernel. I kept hearing that it was very easy to configure and maintain. I had OpenVPN running on my EdgeRouter-X. It was really nice to have it running on my router, because it's always on and connected directly to my modem. If I were going to run Wireguard, I'd want to do it the same way.