<div align="center" markdown="1">

<img src=".github/meshtastic_logo.png" alt="Meshtastic Logo" width="80"/>
<h1>Meshtastic Firmware</h1>

![GitHub release downloads](https://img.shields.io/github/downloads/meshtastic/firmware/total)
[![CI](https://img.shields.io/github/actions/workflow/status/meshtastic/firmware/main_matrix.yml?branch=master&label=actions&logo=github&color=yellow)](https://github.com/meshtastic/firmware/actions/workflows/ci.yml)
[![CLA assistant](https://cla-assistant.io/readme/badge/meshtastic/firmware)](https://cla-assistant.io/meshtastic/firmware)
[![Fiscal Contributors](https://opencollective.com/meshtastic/tiers/badge.svg?label=Fiscal%20Contributors&color=deeppink)](https://opencollective.com/meshtastic/)
[![Vercel](https://img.shields.io/static/v1?label=Powered%20by&message=Vercel&style=flat&logo=vercel&color=000000)](https://vercel.com?utm_source=meshtastic&utm_campaign=oss)

<a href="https://trendshift.io/repositories/5524" target="_blank"><img src="https://trendshift.io/api/badge/repositories/5524" alt="meshtastic%2Ffirmware | Trendshift" style="width: 250px; height: 55px;" width="250" height="55"/></a>

</div>

</div>

<div align="center">
	<a href="https://meshtastic.org">Website</a>
	-
	<a href="https://meshtastic.org/docs/">Documentation</a>
</div>

> **This is a fork.** It tracks [meshtastic/firmware](https://github.com/meshtastic/firmware) and adds MQTT downlink filtering for chatting on busy public channels. The upstream documentation below applies unchanged; see [About this fork](#about-this-fork) for what differs.

## Overview

This repository contains the official device firmware for Meshtastic, an open-source LoRa mesh networking project designed for long-range, low-power communication without relying on internet or cellular infrastructure. The firmware supports various hardware platforms, including ESP32, nRF52, RP2040/RP2350, and Linux-based devices.

Meshtastic enables text messaging, location sharing, and telemetry over a decentralized mesh network, making it ideal for outdoor adventures, emergency preparedness, and remote operations.

### Get Started

- 🔧 **[Building Instructions](https://meshtastic.org/docs/development/firmware/build)** – Learn how to compile the firmware from source.
- ⚡ **[Flashing Instructions](https://meshtastic.org/docs/getting-started/flashing-firmware/)** – Install or update the firmware on your device.

Join our community and help improve Meshtastic! 🚀

## Stats

![Alt](https://repobeats.axiom.co/api/embed/8025e56c482ec63541593cc5bd322c19d5c0bdcf.svg "Repobeats analytics image")

## About this fork

Everything above is upstream documentation and still applies: build and flash the same way, using the same environments. This section covers only the differences.

### Branches

| Branch | Base | Use |
| --- | --- | --- |
| `mqtt-downlink-filter` | release tag `v2.7.26.54e0d8d` | Run this on hardware today. Tested on a Heltec V4. |
| `mqtt-downlink-filter-develop` | upstream `develop` | The same changes ported forward, ready for the next release. Builds clean; not yet tested on hardware. |
| `develop`, `master` | — | Left untouched so they stay clean for pulling from and proposing to upstream. |

**This is the release-tag branch**, the one to flash for day-to-day use.

### The problem it solves

Turning on MQTT downlink for a busy public channel (for example `LongFast` on `mqtt.meshtastic.org`) makes a stock node impractical for chat:

- Every NodeInfo, Position and Telemetry packet on the topic is injected into the node and added to the node database, evicting nodes actually heard over RF.
- Each previously unseen node raises a "New Node Seen" notification on the paired phone. On the public broker that is thousands per day.
- Any downlinked packet that still has hops remaining is rebroadcast over LoRa, so local airtime is spent relaying traffic that arrived from the internet. Neither upstream firmware nor the public broker zeroes hop counts, so this happens by default.
- Most of the stream cannot be used at all. Roughly 98% of packets on the public `LongFast` topic are payload-stripped envelopes (header metadata only) or packets encrypted for someone else's channel of the same name, and the node spends its budget decrypting them.

The goal of this fork is hybrid operation: keep two-way chat over MQTT *and* full RF participation, without flooding the local airwaves or the phone. The node still transmits, relays for its neighbors, and uplinks local RF traffic to MQTT.

### What changed

All changes are in the MQTT receive path (`src/mqtt/MQTT.cpp`) plus guards where the node database is written. Both the phone client proxy and the direct WiFi client feed the same path, so the behavior is identical in either mode.

| Change | Behavior |
| --- | --- |
| Reject empty and foreign envelopes early | Envelopes whose packet carries no payload variant are dropped, as are encrypted packets whose channel hash does not match the local channel matched by name. Happens before packet-pool allocation, decrypt attempts and logging. |
| Drop Position and Telemetry | Telemetry is always discarded. Positions are discarded unless the sender is an established chat partner or already in the node database, so a chat author's location still resolves. Together these are the bulk of the volume. |
| Gate NodeInfo | Accepted only from senders we have received a text message from, or nodes deliberately added to the node database (for example imported contacts). Everything else is dropped. |
| Never create nodes from MQTT | `via_mqtt` packets may refresh a node that already exists but never create one, in `NodeDB::updateFrom` and in the NodeInfo, Position and Telemetry handlers. The phone keeps its own node list, so names still display there. |
| Never rebroadcast MQTT over RF | `hop_limit` is forced to 0 at ingress, so broker traffic cannot be re-aired regardless of the hop count it arrived with, or of broker policy. |
| Solicit NodeInfo from new chat authors | The first text from an unknown author triggers a NodeInfo exchange so their name resolves in seconds instead of waiting for their next periodic broadcast. Sent at hop limit 0, and marked interactive so `allocReply()` applies its 60s throttle rather than the 10 minute periodic one. |
| Log undecryptable ciphertext | Packets the node holds no key for are dumped to the console as `Undecryptable id=… fr=… ch=… len=… data=<hex>`, so a host-side tool can try keys this node does not have (for example to find neighbours still on the stock `AQ==` key). Only failed decrypts are logged, a few lines a minute in a typical area. Set `LOG_UNDECRYPTABLE_CIPHERTEXT` to 0 to compile it out. |
| Subscribe at QoS 0 | Upstream subscribes at QoS 1, which makes the broker queue undelivered messages per client. A node that cannot drain a busy topic overflows that queue and gets dropped as a slow consumer. Mesh traffic is best-effort anyway. |

Chat partners are tracked in a small fixed-size list in RAM (32 entries, cleared on reboot). It is never persisted, so no MQTT-sourced node reaches flash.

Why the console rather than the client API: the API already forwards undecryptable packets with their payload, but every connected client draws from one shared queue, so a phone consumes packets a monitor never sees. The console has no such contention, and the phone keeps working normally.

### Recommended configuration

For chatting on a public broker such as `mqtt.meshtastic.org`:

- `mqtt.enabled = true`, `mqtt.proxy_to_client_enabled = true`. The phone owns the broker connection; see the limitation below before choosing direct WiFi instead.
- `mqtt.encryption_enabled = true` (default).
- Uplink and downlink enabled on the channel you want to chat on.
- `lora.config_ok_to_mqtt = true` so your own packets are allowed onto a public broker.
- `lora.ignore_mqtt = false`. Setting it true drops every `via_mqtt` packet before processing and silences incoming chat entirely.
- Mark the people you chat with as favorites. Favorites are never evicted from the node database.

Two settings in the Android app are easy to confuse. "Proxy to client enabled" is the device field above and requires a reboot. "MQTT proxy on this phone", at the top of the same screen, only starts and stops the relay running on the phone and does not change device config; it has no effect when the device is in direct WiFi mode.

The app has no per-source notification filter, so to silence new-node notifications entirely, turn off the "New node notifications" channel in Android's notification settings for the app. Message notifications use separate channels and are unaffected.

### Measured effect

On the public `LongFast` topic, over identical two-minute windows before and after the early-rejection change:

| | Upstream behavior | This fork |
| --- | --- | --- |
| Packets reaching the decode path | 4102 | 91 |
| Failed decrypt attempts | 4058 | 2 |
| Successful decodes | 46 | 82 |

Text messages published to the topic arrive, decode, and are forwarded to the phone, while no node database entry is created for the sender and nothing is rebroadcast over RF.

### Known limitations

- **Direct WiFi mode is not viable on the public broker.** With the client proxy disabled the node connects and subscribes correctly, but `pubSub.loop()` handles one message per call on a 200 ms scheduler while the topic runs at several hundred messages per second. The node falls behind, the broker drops it as a slow consumer roughly every 30 seconds, and messages published during the gaps are lost. Filtering cannot fix this because the bottleneck is upstream of it, in the socket drain rate. Use the phone client proxy.
- **PKI direct messages require deliberate contacts.** Since MQTT nodes are never auto-added, a DM partner must be imported (QR contact exchange) or favorited before the radio holds the key material needed to decrypt their messages. Broadcast channel chat needs nothing.
- **This node is not an MQTT-to-RF gateway.** Forcing `hop_limit` to 0 means RF-only neighbors will not hear MQTT traffic through it. That is deliberate; it is the airtime cost the fork exists to avoid.
- **Names resolve on the phone, not on the device.** Node names for MQTT contacts live in the app's database. The device screen and node list will not show them.

### Building

Unchanged from upstream. For example, for a Heltec V4:

```bash
pio run -e heltec-v4            # build
pio run -e heltec-v4 -t upload  # build and flash
```

### Staying current with upstream

```bash
git remote add upstream https://github.com/meshtastic/firmware   # once
git fetch upstream
git rebase upstream/master
```

The changes touch few lines in a small number of files, so rebasing onto later releases is normally straightforward.
