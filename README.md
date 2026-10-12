# Link

**Link** is a range of tools for embedded developers, built to understand and
verify links: radio, fieldbus, and the rest. Each one pairs a common base unit
with the interface its job needs. **Link-LR**, the first, speaks LoRa P2P on
868/915 MHz: see [its page](link-lr/README.md).

Its 2.8" touch display makes the tool usable on its own. Unplug it from the
laptop and it keeps working, no phone required. It runs while charging, and a
real-time clock keeps frame timestamps meaningful across a power cycle.

## Repositories

This repository holds what is common to the range: the protocol between the
devices and Link Studio, and the format of the recordings. The rest lives
next door:

| Repository | What it holds |
|---|---|
| [`link-hardware`](https://github.com/semtronik/link-hardware) | Board and enclosure design sources, fabrication exports |
| [`link-firmware`](https://github.com/semtronik/link-firmware) | Device firmware (ESP-IDF) |
| [`link-studio`](https://github.com/semtronik/link-studio) | Desktop application (Tauri) |

## Protocol

Link devices and Link Studio talk JSON over USB, one object per line:

| Document | What it specifies |
|---|---|
| [`protocol/v1/protocol.md`](protocol/v1/protocol.md) | Transport, commands, replies and errors |
| [`protocol/v1/recording.md`](protocol/v1/recording.md) | What a Link-LR records and the format it keeps it in: the same lines live over USB and in `.jsonl` files |

The `v<N>/` folder is the MAJOR version. Backward-compatible evolutions
(added fields) live inside it. A breaking redesign becomes a new `v2/`
alongside `v1/`.

## Semtronik

Link is built by [Semtronik](https://semtronik.com).

## License

MIT — see [LICENSE](LICENSE).
