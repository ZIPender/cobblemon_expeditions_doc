# Cobblemon Expeditions - Documentation

Documentation for [Cobblemon Expeditions](https://modrinth.com/mod/cobblemon-expeditions),
a Fabric and NeoForge mod for Minecraft 1.21.1 that adds timed Pokémon expeditions to
Cobblemon.

This repository holds the docs only. Issues and downloads live on the mod page.

## Guides

| Guide | For |
| --- | --- |
| [Multi-server storage](multi-server.md) | Server owners running a network who want expedition data to follow players between servers. |
| [Custom storage backend API](storage-api.md) | Developers writing an addon that stores expedition data in Redis, Mongo or SQL. |

## Quick answers

**Do I need any of this for a single server?**
No. Everything here is opt-in. A single server or a singleplayer world keeps expedition
data in the world save with no configuration at all.

**My board is empty on the second server.**
That is the default behaviour before you turn sync on: each world save holds its own
copy. See [Multi-server storage](multi-server.md).

**My rank followed me but my Pokémon did not.**
Expected. This mod syncs expedition data, not Pokémon. Party and PC boxes are handled
by your network's own sync layer.

**Can two servers hand out the same reward?**
No. Reward claims and expedition completions go through an atomic claim, so exactly
one server in the network can act on each. If you are writing your own backend, that
is the part to get right - see [the four rules](storage-api.md#the-four-rules).

## Versions

These guides describe Cobblemon Expeditions 1.6.0 and later. Multi-server storage was
added in 1.6.0.
