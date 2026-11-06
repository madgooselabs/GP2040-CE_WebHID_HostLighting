# GP2040-CE Host Lighting — WebHID Tester

A single-file, zero-dependency web page that drives a [GP2040-CE](https://gp2040-ce.info/)
board's RGB LEDs live from the browser, using the **Host Lighting Protocol (HLP) 2.0**
over WebHID. It's a hand-rolled port of the transport in
[djGLiTCH/gp2040ce-hostlighting](https://github.com/djGLiTCH/gp2040ce-hostlighting)
(the reference Python library).

> ⚠️ **Test implementation.** Byte layouts were derived from the project's spec,
> `protocol.py`, and `tests/vectors/vectors.json` — but this has not yet been run
> against real hardware. Treat the first run as verification.

## Requirements

- **Chrome or Edge** (desktop). Firefox/Safari do not implement WebHID.
- A GP2040-CE board with the **Host Lighting add-on enabled** in firmware
  (disabled by default; needs a build with the add-on, e.g. GP2040-CE PR #1691
  or djGLiTCH's pre-release builds).
- The board must be in an input mode that exposes the vendor interface
  (XInput, Keyboard, Generic HID, or SInput). Console modes do **not** carry it.

## Running

Just open `index.html`:

- Double-click it (a `file://` page works for WebHID in Chrome/Edge), **or**
- Serve it over `localhost` if your browser blocks WebHID on `file://`:

```bash
python -m http.server 8000
```

Then browse to `http://localhost:8000/` and click **Connect device**. In the
chooser, pick the GP2040-CE Host Lighting interface (filtered by HID usage page
`0xFF47` / usage `0x0002`).

## Layout

Connection, Device info, and Claim sit in the left column; the Log is pinned
below them (sticky) so it stays in view in both tabs and while you scroll the
longer Advanced panels. The lighting panels are split into two tabs:

- **Basic** — Lighting (fill/clear/stage/commit) and Demo scenarios.
- **Advanced** — Targeted stage (addressing modes), Animation & brightness,
  Events & pages, Keyboard mapping, Raw command, and Reboot / input mode.

## What it does

1. **Connect** → `requestDevice` filtered on usage page `0xFF47`, then auto-sends
   `HELLO` and `GET_PAGE` (summary).
2. **HELLO** → reads magic `GPHL`, protocol version, board ID, queue limit.
3. **GET_PAGE (summary, page 8)** → LED extent, LED limit, render rate, color
   format, and grid size. The LED grid is sized from the reported extent.
4. **CLAIM** → takes the lighting with a random 32-bit token and your app name
   (exclusive by default). Stores the returned generation for later requests.
5. **CONFIGURE** → whole-frame vs overlay takeover, keepalive timeout.
6. **FILL / CLEAR / STAGE grid / COMMIT** → paint swatches with the color picker,
   then stage LED runs (≤19 RGB pixels per 64-byte report) and commit.
7. **Rainbow / Chase** → simple streaming demos.
8. **Auto-keepalive** → sends `HELLO` with the keepalive flag every 750 ms so the
   board doesn't fall back to onboard animations (2 s default timeout).
9. **Animation & brightness** → `SET_ANIMATION` (index, or Off = `0xFFFF`),
   `SET_ANIMATION_SPEED` (idle/pressed/case; blank = leave unchanged = `0xFF`),
   `SET_BRIGHTNESS` (step, or `0xFF` = saved step; optional Save), and
   `SET_PROFILE` (number). These apply live and are saved to flash. **Note:** only
   `SET_ANIMATION` is confirmed against a repo test vector; the other three come
   from the library's method bodies + spec with no on-wire vector, so watch the
   reply status when you run them.
10. **Raw command** → craft any command/flags/payload by hand and see the reply.
11. **Demo scenarios** → one-click test patterns (see below).
12. **Events & pages** → `SUBSCRIBE` (event mask; bit 0 = STATE_CHANGED) and a
    generic `GET_PAGE` for any page number (1 state, 2 modes, 8 summary, 9
    profiles, 10 pins, 11 keymap, 13 animations) — reply logged in hex.
13. **Keyboard mapping** → `SET_KEYBOARD_MAPPING` (profile, first record, then a
    list of HID usage codes). Count is derived from the code list (1–57). Requires
    "Allow hosts to change the keyboard mapping" in firmware + an active profile.
14. **Reboot / input mode** (danger) → `REBOOT` (gamepad / web configurator / UF2
    bootloader) and `SET_INPUT_MODE`. Both reboot the board, carry the required
    magic (`BOOT` / `MODE`), and ask for confirmation first.
15. **Log** → every TX/RX/event frame in hex, color-coded. **Copy** or
    **Save .txt** exports the log with a device-info header (product, VID:PID,
    protocol, board ID, summary) — paste it into the PR when reporting what you saw.

## Demo scenarios (test cases)

Claim first, then click a demo. **Static/pattern** demos stage once and commit;
**animated** demos stream until you hit **Stop**.

| Demo | What it does |
|---|---|
| Red / Green / Blue / White | Solid primary — check the color channels map correctly |
| Picker color | Solid in the current color-picker color |
| Off | `CLEAR` + commit — all LEDs dark |
| Rainbow | Hue spread across the chain by index |
| Gradient | Picker color fading to black along the chain |
| Alternate | Every other LED on/off — checks even/odd addressing |
| Halves | First half bright, second half dim |
| Index map | Each LED a distinct hue by its index — confirm physical ordering |
| Blink ▶ | Whole chain toggling color/off |
| Chase ▶ | A single lit LED walking the chain |
| Breathe ▶ | Whole chain fading up and down |
| Hue cycle ▶ | Whole chain sweeping through hues together |
| Theater ▶ | Every third LED lit, marching (theater-chase) |
| Walk ord ▶ | Lights each light **by ordinal**, one at a time, white (hlp-spice2x phase 2) |

These are host-side patterns staged over the normal `STAGE`/`COMMIT` path, so they
exercise the same code the PR touches — a good quick smoke test of a build.

## Addressing modes (Targeted stage)

The grid and most demos stage by raw **LED run** (addressing 0). A real host like
[hlp-spice2x](https://github.com/djGLiTCH/hlp-spice2x) also stages by other
addressing modes, which the PR's STAGE handler must resolve. The **Targeted stage**
panel exercises them:

| Addressing | Address field means | hlp-spice2x maps from |
|---|---|---|
| 1 light run | first light ordinal (+count) | — |
| 4 light | one light ordinal | a profile entry's `index` |
| 5 pin | a page-10 pin key | — |
| 6 action | a `GpioAction` (signed i16) | a control/`button` name (B1, UP, S2…) |
| 7 kind | a `LightType` (PLED1-4, CASE, TURBO…) | a `button` kind name |

Entry modes (4/5/6/7) use the entry format `[3]` addr|fmt, `[4]` count, then
`[address][RGB]` per entry — **action is a signed 16-bit value**. Get real action /
kind / ordinal numbers from a page read or the bridge's `--list-lights`.

**Precedence (phase 7):** reproduces the hlp-spice2x `hardware_lights` precedence
check on a control that has two lights — control red → that one light cyan (the
per-light entry overrides the control) → that light black (overrides even to off)
→ cleared (both red again). Fill in the control's `GpioAction` and one of its light
ordinals, then run it.

## Testing a PR with it

Each tester flashes the candidate firmware build, opens `index.html`, clicks
through Connect → Claim → Fill/Stage/Commit and watches the board, then uses
**Copy**/**Save .txt** to attach the session log (with device fingerprint) to the
PR discussion. The hex log makes it easy to pin down exactly which frame behaved
differently between builds.

## Protocol notes (as implemented)

| Thing | Value |
|---|---|
| Discovery | usage page `0xFF47`, usage `0x0002` (do **not** match VID/PID) |
| Report | unnumbered, 64 bytes in + 64 bytes out |
| Framing | `[0]` command, `[1]` sequence, `[2]` flags, `[3..63]` payload (LE) |
| Reply | `cmd \| 0x80`, status in `[2]` (0 OK, 1 unsupported, 2 invalid, 3 stale, 4 claimed) |
| Event | starts with `0x80`, event number in `[1]` |
| Flags | `0x01` commit-after, `0x02` no-reply, `0x04` keepalive, `0x08` holder, bits 4-6 generation |
| STAGE run payload | `[3]` addressing\|(format<<4), `[4..5]` first LED (LE u16), `[6]` count, `[7..]` pixels |
| Pixels | always sent R,G,B; board converts to its chain format (GRB/RGB/GRBW/RGBW) |

### WebHID vs hidapi

The reference library (hidapi) prepends a leading `0x00` report-ID byte and uses
65-byte buffers. **WebHID does not** — you call `device.sendReport(0, <64 bytes>)`
and the browser handles the report ID. So this code sends/receives plain 64-byte
reports with no leading byte. The hex in the repo's `vectors.json` is also written
without that leading byte, so it lines up with what you'll see in the log here.

## Known gaps / to verify against hardware

- The GET_PAGE summary parse assumes the record area begins at byte 12
  (`HEADER_RECORDS`) with `led_extent`/`led_limit`/`render_rate` as u16 at
  `+12/+14/+16`, color format `+18`, flags `+19`, grid `+26/+28`. Matches the
  `led_limit` vector; confirm on your board.
- `SET_ANIMATION`, `SET_ANIMATION_SPEED`, `SET_BRIGHTNESS` and `SET_PROFILE` are
  wired up. `SET_ANIMATION` and `CONFIGURE` match repo test vectors; the speed /
  brightness / profile layouts come from the library's method bodies + spec but
  have no on-wire vector — watch the reply status when exercising them.
- `SET_KEYBOARD_MAPPING` and `SUBSCRIBE` are wired up and vector-confirmed.
  `REBOOT` and `SET_INPUT_MODE` are source-only (no on-wire vector) and reboot the
  board — they're behind confirmation dialogs.
- Whether Chrome accepts a usage-only WebHID filter (no VID/PID) is assumed. If
  the chooser shows nothing, add a VID/PID filter — the project's udev rule lists
  `24c6:5bad`, `cafe:0001`, `10c4:82c0`, `2e8a:10c6` as examples (mode-dependent,
  user-overridable).
