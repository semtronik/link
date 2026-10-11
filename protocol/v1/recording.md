# Recording

What a Link-LR keeps of what its radio sees, and the format it keeps it
in. The same lines travel live over USB (see [protocol.md](protocol.md)):
one format, one parser, whether the app is reading a device or a file.

## Principles

- **Always on.** Recording starts at power-up and never stops. There is no
  start or stop control.
- **The device is the memory.** Link Studio keeps nothing between two
  openings: on connection it reads what the device holds. Whatever must
  outlive the device's capacity is exported to a file.
- **Bounded, oldest first out.** Recordings take at most 4 MiB of the data
  partition. Past that, the oldest file is deleted. The partition itself
  stays at its full size, so wear levelling spreads over all of it.
- **Written in blocks.** Lines are flushed to flash when 4 KiB are pending
  or every 2 s, whichever comes first. A hard power cut loses at most the
  last 2 s.
- **Gaps are shown, never filled.** Every interruption is either a line of
  its own or the space between two sessions, with its reason.
- **The clock must be set.** No session opens while the clock is not
  valid; the Journal screen says so.

## Sessions

A session is a stretch during which the radio was not touched: every
line in it was received or sent with the same settings, so statistics
over a session never mix two configurations.

A session **closes** on:

| Reason | Event |
|---|---|
| `config` | A parameter validated in Config, changed or not |
| `profile` | Another profile selected |
| `scan` | The band sweep started (it borrows the radio) |
| `clear` | The clear button of the Journal action bar |
| `app` | A setting command received from the app |
| `time` | The clock set, from the device or the app |
| `erase` | The recordings erased |
| `shutdown` | Power-off from the button |
| `battery` | Power-off on critical battery |
| `reboot_cmd` | Restart requested by the app |

A session **opens** when the Journal screen is displayed and no session
is running. The device starts on the Journal, so power-up opens one.
Other screens (preferences, date and time, about...) leave the session
alone. On a first start, or after a factory reset, the setup (language,
region, clock) comes before the Journal, and the clock step alone when
only the clock was lost: nothing is recorded until it is done.

A reset or a crash leaves no closing line. The next session's boot reason
says what happened.

## Storage

- Each session starts a new file. A file holds at most 128 KiB; a longer
  session continues in the next one.
- Every file starts with the session's opening line, so a session whose
  first files were deleted by rotation still says what it was recorded
  with.
- Rotation deletes whole files, oldest first.
- A factory reset deletes them all, with no closing line
  ([protocol.md](protocol.md), `factory_reset`).

File names and layout on the partition are internal. The app goes
through the commands in [protocol.md](protocol.md), never through files.

## Format

JSON Lines: one object per line, UTF-8, `\n`-terminated. Every object has
a `type`.

Conventions:

- A field with no value is **omitted**, never set to 0 or `null`.
- Units are in the field name where there is one: `_hz`, `_dbm`, `_db`.
- `ts` is wall-clock time, milliseconds since 1970-01-01 UTC. It only
  dates a line: it can move when the clock is corrected. Line order is
  file order.

### `session`

First line of a session, and of each of its files.

| Field | Type | Content |
|---|---|---|
| `v` | int | Format version, currently 1 |
| `id` | int | Session number, unique among the sessions on the device |
| `part` | int | File index within the session, from 1 |
| `ts` | int | When the session opened |
| `open` | string | Why: `boot`, or the reason the previous session closed |
| `boot` | string | When `open` is `boot`: `power_on`, `reset_pin`, `software`, `panic`, `watchdog`, `brownout`, `usb` (reset from the USB port, e.g. flashing), `jtag`, `other` |
| `fw` | string | Firmware version |
| `dev` | string | Device id, its MAC address |
| `profile` | object | The settings, below |

`profile`, every radio setting the session ran with. Region and receiver
gain belong to the profile like the rest:

| Field | Type | Content |
|---|---|---|
| `name` | string | Profile name: `Profile 1` to `Profile 3` for the base profiles, whatever the device's language; the name given by Link Studio for an imported one |
| `region` | string | `EU868` or `US915` |
| `freq_hz` | int | Carrier |
| `sf` | int | Spreading factor, 5 to 12 |
| `bw_hz` | int | Bandwidth |
| `cr` | int | Coding rate denominator, 5 to 8 for 4/5 to 4/8 |
| `power_dbm` | int | Transmit power, conducted |
| `preamble` | int | Preamble length, symbols |
| `crc_on` | bool | CRC sent with our frames |
| `iq_inverted` | bool | Inverted IQ, as LoRaWAN downlinks; false is standard |
| `sync_word` | int | 18 (0x12) private, 52 (0x34) public |
| `rx_gain` | string | `saving` (chip default) or `boosted` |

### `end`

Last line of a session, when it closed cleanly.

| Field | Type | Content |
|---|---|---|
| `ts` | int | When it closed |
| `reason` | string | See [Sessions](#sessions) |

### `rx`

One reception, successful or not.

| Field | Present for | Content |
|---|---|---|
| `ts` | all | The frame's first symbol, computed back from the end of the frame when its length is known, otherwise from its header |
| `status` | all | `ok`, `crc_err` (CRC failed), `header_err` (header failed its own check), `incomplete` (a header passed, nothing complete followed) |
| `len` | `ok`, `crc_err` | Payload length, bytes |
| `data` | `ok`, `crc_err` | Payload, hex. Corrupt when `status` is `crc_err` |
| `rssi_dbm` | `ok`, `crc_err` | Signal and noise together, averaged over the frame |
| `signal_dbm` | `ok`, `crc_err` | The signal alone after despreading (SignalRssiPkt) |
| `snr_db` | `ok`, `crc_err` | Quarter dB steps, e.g. `-5.75` |
| `freq_err_hz` | `ok`, `crc_err` | Sender's offset from the tuned frequency, signed |
| `toa_ms` | `ok`, `crc_err` | Time on air as the device computed and shows it, from the frame's length, coding rate and CRC with the session's other settings; truncated to the ms |
| `cr` | `ok`, `crc_err`, `incomplete` | From the frame's header |
| `crc_on` | `ok`, `crc_err`, `incomplete` | From the frame's header |
| `noise_dbm` | when sampled | The channel just before the frame: median of the last 20 instantaneous RSSI readings taken between frames, one every 100 ms (2 s) |
| `inst_n` | when sampled | Instantaneous RSSI readings taken during the frame, from the preamble to its end, one every 10 to 20 ms: their number follows the frame's length |
| `inst_min_dbm`, `inst_max_dbm` | when sampled | The lowest and highest of those readings |

`noise_dbm` is the background noise where the device stands, plus
whatever transmits continuously; a short burst barely moves it. Set
against `signal_dbm`, it says how far a frame stood above the noise.
Compared across places or times, it shows where the channel is quieter.
A noisy environment raises it, and so does the device itself when it is
powered over USB, by about 3 dB on our bench.

`inst_*` say whether the channel stayed steady while the frame was being
received. A peak well above the frame's own level can mean that
something else was transmitting at the same time, so that the frame was
disturbed; a frame that failed without such a peak points rather at a
signal too weak. When the signal is below the noise, the readings show
the noise, not the frame: compare the peak with `signal_dbm` or
`noise_dbm`, whichever is higher. A burst shorter than the interval
between readings can go unseen. The instantaneous RSSI is not calibrated
(SX1261/2 datasheet, 6.1.6): compare readings within a frame or a device,
not across devices.

### `tx`

One of our transmissions.

| Field | Type | Content |
|---|---|---|
| `ts` | int | Start |
| `len` | int | Payload length, bytes |
| `data` | string | Payload, hex |
| `toa_ms` | int | Time on air as the device computed and shows it, truncated to the ms |
| `result` | string | `ok`, `timeout` (TX done never came), `aborted` (cut short by a reconfiguration or the sweep) |

### `gap`

Reception interrupted inside a session.

| Field | Type | Content |
|---|---|---|
| `from` | int | When it stopped |
| `to` | int | When it resumed |
| `reason` | string | `radio_reset`: the chip stopped answering and was reset |

A transmission also interrupts reception; its `tx` line already says so.

### `lost`

Lines the device produced but could not keep, because writing fell
behind. Should never appear.

| Field | Type | Content |
|---|---|---|
| `ts` | int | When the loss was noticed |
| `count` | int | Lines lost |

## Sizes

Estimates, not measurements: an `rx` line for a 14-byte frame is about
250 bytes; a 255-byte frame about 730. At one short frame a minute,
4 MiB holds about 11 days.
