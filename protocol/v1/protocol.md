# Protocol

How a Link device and Link Studio talk over USB. Written from the
Link-LR firmware. `tools/link_rec.py` in link-firmware drives the
recording and profile commands from a command line.

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

### `factory_reset`

| Field | Content |
|---|---|
| `confirm` | Must be `"erase"`; anything else is refused |

The device as it left the bench, firmware aside. `ack`, then half a
second later the clock is left unset, the product's own data (Link-LR:
every recording), the event log and every setting are erased, and the
device restarts on its first-start setup. Nothing is recorded of it:
there is nowhere left to record it.

The same reset is on the device, at the bottom of the settings screen,
behind a confirmation.

| Error `msg` | Meaning |
|---|---|
| `confirm must be "erase"` | `confirm` missing or wrong; nothing erased |

## Link-LR commands

Every radio setting belongs to the selected profile, whose object is the
`profile` of the `session` line ([recording.md](recording.md)).

### `get_profile`

Reply: `ack` with `profile`, the selected profile. Leaves the session
alone.

### `set_profile`

| Field | Content |
|---|---|
| `profile` | Object: the fields to change, named as in the session line's `profile` |

Fields left out keep their value. Every field is checked before anything
is touched: one bad value leaves the radio, the profile and the session
as they were. Then the settings are applied to the radio and stored in
the selected profile, and the current session closes (reason `app`),
whether or not a value changed.

| Field | Accepted |
|---|---|
| `region` | `"EU868"` or `"US915"`. Informs, never restricts: it only changes how transmissions are assessed against the regulations |
| `freq_hz` | 862000000 to 930000000 |
| `sf` | 5 to 12 |
| `bw_hz` | 62500, 125000, 250000 or 500000, the choices of the config screen |
| `cr` | 5 to 8 |
| `power_dbm` | -9 to 22 |
| `preamble` | 1 to 65535 |
| `crc_on`, `iq_inverted` | `true` or `false` |
| `sync_word` | 0 to 255 |
| `rx_gain` | `"saving"` or `"boosted"` |

Numbers must be whole. Reply: `ack` with `profile`, as applied.

| Error `msg` | Meaning |
|---|---|
| `profile must be an object` | No `profile`, or not an object |
| `<field> must be ...` | That value is not accepted |
| `name cannot be set` | A profile's name is not the device's to change |
| `unknown field: <field>` | Not a profile field; nothing is ignored silently |
| `no radio module` | No radio fitted or answering |
| `radio refused the settings` | The chip did not take them; nothing stored |

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
`session`, `end`, `rx`, `tx`, `gap`, `lost`. They are pushed whether or not
the app asked.

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

## Replaced

| Gone | Replaced by |
|---|---|
| `pkt` | the `rx` line |
| `txd` | the `tx` line |
| `cfg`, `get_config` | the `session` line, and `get_profile` |
| `set_freq`, `set_sf`, `set_bw`, `set_rx_gain`, `set_region` | `set_profile`: one command, one session closed, however many settings change |
