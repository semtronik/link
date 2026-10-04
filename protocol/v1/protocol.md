# Protocol

How a Link device and Link Studio talk over USB. Written from the
Link-LR firmware; the recording commands and messages are specified but
not yet implemented, see [Changes planned](#changes-planned).

## Transport

- USB CDC (the ESP32-S3's USB Serial/JTAG port).
- One JSON object per line, `\n`-terminated, both ways.
- The device's log output shares the port. Only lines starting with `{`
  are messages; everything else is to be ignored.
- A line sent to the device is at most 6 KiB; a longer one is cut and
  rejected as malformed.

## Commands and replies

A command is `{"cmd": "<name>", ...}`. Its argument is `value` unless
stated otherwise.

Replies:

| Reply | Shape |
|---|---|
| Success | `{"type":"ack","cmd":"<name>","status":"ok"}`, or an `ack` carrying the requested data |
| Failure | `{"type":"err","cmd":"<name>","msg":"<why>"}` |
| Malformed JSON | `err` with `cmd` set to `"?"` |
| Unknown command | `err` with `msg` `"unknown command: <name>"` |

Commands that stream several lines bracket them between two `ack`s with
`"phase":"begin"` and `"phase":"end"`, the latter carrying `count`. The
app knows the stream is complete, not merely paused.

## Platform commands

Common to every Link product.

### `get_version`

Reply: `ack` with `product`, `model`, `version`, `build_date`.

`product` and `model` are currently `"LFT"` and `"v1"`, the strings of
the previous product, because the app recognises no other pair yet.

### `get_time`

Reply: `{"type":"time","utc":<epoch s>,"offset_min":<int>,"valid":<bool>}`.
`valid` is false while the clock has not been set.

### `set_time`

| Field | Content |
|---|---|
| `utc` | Epoch seconds, 2024-01-01 to 2100-01-01 |
| `offset_min` | Optional, local offset from UTC, -720 to 840 |

Writes the real-time clock and the system clock. Closes the current
session (reason `time`).

### `get_events`

The device's event log (boots, shutdowns, clock set, firmware updates),
streamed: `begin`, then one `{"type":"event","line":"<text>","seq":<n>}`
per line, then `end` with `count`.

### `ota_begin`, `ota_data`, `ota_end`

Firmware update.

| Command | Fields | Reply |
|---|---|---|
| `ota_begin` | `size` (bytes), `sha256` (hex) | `ack` with `partition` |
| `ota_data` | `data`: a chunk, base64 | `ack` with `written`, total bytes so far |
| `ota_end` | | `ack` with `status` `"success"` once the image is verified |

Any failure aborts the whole update; it restarts from `ota_begin`.

### `reboot`

`ack`, then the device restarts half a second later. Closes the current
session (reason `reboot_cmd`).

## Link-LR commands

### `get_config`

Reply: `{"type":"cfg","freq":<MHz>,"sf":<int>,"bw":<kHz>,"drops":<int>,"rx_gain":"saving"|"boosted","region":"EU868"|"US915"}`.
`drops` is the number of frames whose CRC failed since boot.

### `set_freq`, `set_sf`, `set_bw`

| Command | `value` |
|---|---|
| `set_freq` | MHz, 862 to 930 |
| `set_sf` | 5 to 12 |
| `set_bw` | kHz: 125, 250 or 500 |

Applied to the radio at once and stored. Each closes the current session
(reason `app`).

### `set_rx_gain`

`value`: `"saving"` (the chip's default) or `"boosted"`. Closes the
current session (reason `app`).

### `set_region`

`value`: `"EU868"` or `"US915"`. Informs, never restricts: it only
changes how transmissions are assessed against the regulations.

### `tx`

`value`: the payload, 1 to 255 bytes in hex. `ack` once the frame is on
air; how it ended follows as a `tx` line.

| Error `msg` | Meaning |
|---|---|
| `no radio module` | No radio fitted or answering |
| `radio busy (scan)` | The band sweep holds the radio |
| `previous frame still on air` | One transmission at a time |

## Recording

What is recorded and in what format is in [recording.md](recording.md).

### Live lines

Every line written to the recording is also sent as it is produced:
`session`, `end`, `rx`, `tx`, `gap`, `lost`. They are pushed whether or
not the app asked.

### `get_sessions`

The sessions on the device, oldest first, streamed: `begin`, then one
line per session, then `end` with `count`.

| Field | Content |
|---|---|
| `type` | `"session_info"` |
| `id` | Session number |
| `ts` | When it opened |
| `end_ts` | Its last line's `ts`; absent if it has no lines yet |
| `reason` | Why it closed; absent if it is running or ended without a closing line |
| `current` | `true` for the running session |
| `truncated` | `true` when rotation deleted its first files |
| `bytes` | Its size on the device |

### `get_session`

| Field | Content |
|---|---|
| `id` | Session number |

`begin` with `id`, then the session's lines exactly as recorded, then
`end` with `id` and `count`. Error `msg` `unknown session` if it is not
on the device.

For the running session, the stream covers every line up to `end`,
including those not yet flushed to flash. Live lines produced meanwhile
are held and sent after `end`, so reading the running session and then
following it live leaves neither a gap nor a duplicate. Live lines that
arrived before `begin` are also in the stream and must be ignored.

### `erase_sessions`

Deletes every recording. Closes the current session (reason `erase`); a
new one opens as soon as the Journal screen is displayed. `ack` once
done.

### On connection

1. `get_sessions`.
2. `get_session` on the running one, then follow the live lines.
3. Past sessions on demand, with `get_session`.

## Changes planned

With recording:

- `pkt` (good frames only) is removed. The `rx` line replaces it.
- `txd` is removed. The `tx` line replaces it.
- `cfg` and `get_config` are replaced by the `session` line, which
  carries the whole profile.

With the profile carrying every radio setting, gain and region included:

- `set_freq`, `set_sf`, `set_bw`, `set_rx_gain` and `set_region` are to
  be revised. One command per parameter closes one session per
  parameter; a single command setting the whole profile would close one.
