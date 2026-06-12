# Torrent Analyze

## Overview

Someone on the company network was torrenting files, and I had a packet capture to figure out what they downloaded. The flag format told me the answer would be a filename, so my goal was to identify the exact file being torrented.

## Analysis

I opened the capture in Wireshark and did a quick scan of what protocols were present. Most of it was TCP and TLS traffic, but there were also DNS queries and a chunk of UDP traffic. The DNS queries immediately caught my eye since they were going to `torrent.ubuntu.com` and `ipv6.torrent.ubuntu.com`. That confirmed someone was using a BitTorrent client, and based on the domain, they were probably downloading something Ubuntu related.

My first thought was to look at the HTTPS traffic and try to pull a `.torrent` file from the Ubuntu tracker. That idea died quickly since the traffic was TLS encrypted and I had no key to decrypt it.

I pivoted to the UDP traffic instead. I started following individual UDP streams by right-clicking packets and selecting Follow > UDP Stream. What I kept seeing in these streams looked like this:

```
d1:ad2:id20:...AK..w]}..hf...:..9:info_hash20:.F|.....A6{."0....X.e1:q9:get_peers...
```

The `d1:ad2` at the start was unfamiliar at first, but searching for it online confirmed that is the standard beginning of a BitTorrent DHT payload. The protocol these packets were using is called Mainline DHT, which is how BitTorrent clients find peers without relying on a central tracker. Instead of connecting directly to a tracker, a client broadcasts `get_peers` queries across a distributed hash table, using a unique identifier called an `info_hash` to locate other people sharing the same file.

That `info_hash` is exactly what I needed. It is a SHA-1 hash that uniquely identifies a torrent, and looking it up on Google typically reveals the filename associated with it.

To find the hash, I used Wireshark's search feature. I went to Edit > Find Packet, switched the search type to "Packet Bytes", and searched for the string `info_hash`. That brought me to the relevant packets. Looking at the hex dump of one of the matching packets:

```
0050   5f 68 61 73 68 32 30 3a e2 46 7c bf 02 11 92 c2   _hash20:.F|.....
0060   41 36 7b 89 22 30 dc 1e 05 c0 58 0e 65 31 3a 71   A6{."0....X.e1:q
```

The 20 bytes immediately following `info_hash20:` are the raw hash. Reading those bytes out gives:

```
e2 46 7c bf 02 11 92 c2 41 36 7b 89 22 30 dc 1e 05 c0 58 0e
```

Which translates to the hex string: `e2467cbf021192c241367b892230dc1e05c0580e`

I dropped that hash straight into Google. Multiple results came back pointing to the same file: `ubuntu-19.10-desktop-amd64.iso`. The hash matched the official Ubuntu 19.10 desktop ISO, which lined up perfectly with the DNS queries I had seen at the start going to the Ubuntu torrent tracker.



## Flag

```
picoCTF{ubuntu-19.10-desktop-amd64.iso}
```



## Key Takeaways

BitTorrent DHT traffic is usually visible in plaintext even when the surrounding traffic is encrypted, because the peer discovery protocol does not use TLS. The `info_hash` embedded in `get_peers` messages is all you need to identify what is being torrented. A simple search on Wireshark for the string `info_hash` in packet bytes is enough to locate it, and the hash itself is publicly searchable.
