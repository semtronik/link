# Link

**Link** is a modular field toolkit: a common ESP32-S3 base unit with a
display, paired with an interchangeable interface module, in a rugged
enclosure. One device, one desktop application — the interface module decides
what it talks to.

> **Status: pre-production.** Hardware is under validation on our own bench.
> Nothing is shipping to customers yet.

## The range

| Product | Interface | Status |
|---|---|---|
| **Link-LR** | LoRa P2P, 868/915 MHz | first product, in development |
| **Link-485** | RS485 | planned |

Every product shares the same base unit, the same enclosure footprint and the
same application. Adding an interface means adding a module, not a new
product line.

## Repositories

| Repository | What it holds |
|---|---|
| [`link-hardware`](https://github.com/semtronik/link-hardware) | Board and enclosure design sources, fabrication exports |
| [`link-firmware`](https://github.com/semtronik/link-firmware) | Device firmware (ESP-IDF) |
| [`link-studio`](https://github.com/semtronik/link-studio) | Desktop application (Tauri) |
| this one | Product overview, device↔app protocol, user manual |

## Protocol

Link devices and Link Studio talk over a JSON protocol, specified here as
versioned JSON Schemas plus a readable document:

```
protocol/
└── v1/
    ├── *.schema.json
    └── examples/
```

The `v<N>/` folder is the MAJOR version. Backward-compatible evolutions
(added fields) live inside it; a breaking redesign becomes a new `v2/`
alongside `v1/`.

Firmware and application keep independent implementations, but both CIs
validate the shared example messages against these schemas — divergence
breaks the build.

Once devices are in customers' hands, the published protocol version is a
contract: third parties can build their own clients on it.

## Semtronik

Link is built by [Semtronik](https://semtronik.com).
