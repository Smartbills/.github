# Xbox 360 Controller Bridge — Feasibility & Architecture

**Status:** Feasibility accepted — ready to spike
**Goal:** Use a JTAG/RGH Xbox 360 console as a wireless receiver so its controllers appear on Windows as native Xbox 360 gamepads, with no additional hardware purchase.

---

## 1. Verdict

**Yes, this is possible**, and the two hardest-looking pieces already exist as open source:

1. **The console can already read its wireless controllers from homebrew.** libxenon's USB controller driver (`libxenon/drivers/usb/usbctrl.c`) explicitly detects the console's *internal RF module* and treats it as a 4-slot wireless receiver — not just wired pads. It matches on product IDs `0x0291` (internal RF module), `0x02a9` and `0x02aa` (third-party receivers), allocates four controller instances per receiver, parses the wireless packet format (`b[5] == 0x13` marks a state packet; sticks at bytes 6–13, triggers at 4–5, buttons bitfielded in 2–3), and handles pad disconnection (`b[0] == 0x08 && b[1] == 0x00`). Rumble is already implemented via `usbctrl_set_rumble()` / `set_controller_rumble()`.
2. **Windows can already accept a synthetic Xbox 360 pad.** [ViGEmBus](https://github.com/nefarius/ViGEmBus) is a **signed** kernel-mode driver that materialises a virtual XUSB device. Games see a genuine Xbox 360 controller through XInput — no per-game work, no shims, and critically **no driver signing costs for us**.

So the project is not "write a Windows driver and reverse-engineer a radio protocol". It is "write a transport between two solved endpoints". That is a materially smaller and lower-risk piece of work.

### Scope gate — read this first

This project is **only worth building for wireless controllers.** A *wired* Xbox 360 controller already works on Windows out of the box via the in-box XUSB driver. If the pads on hand are wired, there is no project.

### Honest caveat on "no additional hardware"

It is true we buy nothing. But the runtime cost is that **the Xbox 360 must be powered on and on the LAN** every time you want to play. Compared to a wireless receiver dongle, this trades ~$15 for a full console's desk space, power draw, and boot time. Worth building because we own the hardware and it is a genuinely interesting piece of engineering — not because it is operationally cheaper.

---

## 2. Architecture

```
┌─────────────────────── Xbox 360 (JTAG/RGH) ────────────────────────┐
│  Wireless pads ──2.4GHz──> internal RF module (USB PID 0x0291)     │
│                                     │                              │
│                          libxenon usbctrl.c                        │
│                                     │                              │
│                       bridge.xex  (our code)                       │
│                    poll 4 pads → pack → lwIP UDP                   │
└─────────────────────────────────────┬──────────────────────────────┘
                                      │  Ethernet / LAN
                                      │  UDP :27360  state   (360 → PC)
                                      │  UDP :27361  rumble  (PC → 360)
┌─────────────────────────────────────┴──────────────────────────────┐
│  X360Bridge.Agent  (.NET 9, Windows tray app / service)            │
│      UDP listener → decode → Nefarius.ViGEm.Client                 │
│                                     │                              │
│                             ViGEmBus (signed KMDF)                 │
│                                     │                              │
│                   4 × virtual XUSB pads → XInput → games           │
└────────────────────────────────────────────────────────────────────┘
```

### 2.1 Console side — `bridge.xex`

Built with **[libxenon](https://github.com/Free60Project/libxenon)** (open source, clean-room, no leaked XDK — keeps us legally clean). Bare-metal PowerPC, launched from XeLL / Aurora / XeXMenu.

Main loop:

```c
network_init();                 // libxenon: lwIP + DHCP
usb_do_poll();                  // pump USB, fills controller state
for (int i = 0; i < 4; i++) {
    struct controller_data_s c;
    get_controller_data(&c, i);
    if (changed(i, &c)) send_state(i, &c);
}
network_poll();                 // pump lwIP
```

**Networking note:** `network.h` exposes only `network_init()`, `network_poll()` and `network_print_config()`, and pulls in `<lwip/netif.h>` / `<lwip/dhcp.h>` — it does **not** expose BSD sockets. libxenon's lwIP is almost certainly built `NO_SYS=1`, so plan on the **lwIP raw UDP API** (`udp_new`, `udp_bind`, `udp_sendto`, `udp_recv`) driven from the main loop, not `socket()`/`sendto()`. Do not design around BSD sockets until a spike proves `LWIP_SOCKET` is compiled in.

**Bare-metal caveat:** a libxenon `.xex` takes over the console. The bridge cannot run alongside the dashboard — while bridging, the 360 is a dedicated receiver and nothing else. That is acceptable for the use case, but it is not a background service.

### 2.2 PC side — `X360Bridge.Agent`

.NET 9 Windows tray application (Worker Service host), referencing **[`Nefarius.ViGEm.Client`](https://www.nuget.org/packages/Nefarius.ViGEm.Client/)** (current: 1.21.x).

- One `IXbox360Controller` per connected physical pad, created on first packet and `Disconnect()`ed on pad-disconnect or link timeout, so slots mirror the console 1:1.
- Decode packet → `SetButtonState` / `SetAxisValue` / `SetSliderValue` → `SubmitReport()`.
- Subscribe to `FeedbackReceived` to catch large/small motor values and LED index from games, and forward them back to the console on the rumble port.

**ViGEmBus status — known and accepted.** The project was archived in November 2023 following a trademark dispute; the driver still works, is still signed, and still backs DS4Windows and DualSenseX. A successor (`VirtualPad`) is in development. We should isolate all ViGEm calls behind a small `IVirtualPadSink` abstraction so swapping backends later is a one-class change.

### 2.3 Wire protocol

Fixed 20-byte state datagram. **Endianness is the trap here:** Xenon is big-endian PowerPC, x86 is little-endian. Declare the wire format **little-endian** (matching `XINPUT_GAMEPAD` so the PC side is a straight cast) and byte-swap on the console.

| Offset | Size | Field                                              |
|-------:|-----:|----------------------------------------------------|
| 0      | 2    | Magic `0x3658` ("X6")                              |
| 2      | 1    | Protocol version                                   |
| 3      | 1    | Pad index (0–3)                                    |
| 4      | 4    | Sequence number (drop out-of-order, detect loss)   |
| 8      | 2    | Buttons bitfield (XInput layout)                   |
| 10     | 1    | Left trigger                                       |
| 11     | 1    | Right trigger                                      |
| 12     | 8    | ThumbLX, ThumbLY, ThumbRX, ThumbRY (int16 ×4)      |

Rumble datagram (PC → console): magic, version, pad index, large motor, small motor, LED index.

Design rules:
- **UDP, not TCP.** A dropped input frame must be superseded by the next one, never retransmitted — a late-but-correct stick position is worse than a lost one. The sequence number exists to *discard* stale frames.
- **Send on change + a ~100 ms keepalive**, so the PC can detect a dead link and disconnect the virtual pad rather than leaving a stick jammed.
- **Version the header from day one.**

### 2.4 Latency budget

| Stage                          | Estimate  |
|--------------------------------|-----------|
| RF poll interval (360 wireless)| 4–8 ms    |
| Console pack + lwIP TX         | ~1 ms     |
| LAN (wired switch)             | 0.3–1 ms  |
| Agent decode + `SubmitReport`  | ~1 ms     |
| ViGEmBus → XInput → game poll  | 1–8 ms    |
| **Added vs. a direct dongle**  | **~8–15 ms** |

Fine for single-player, platformers, racing, couch co-op. Measurably worse for competitive FPS and fighting games. **Use wired Ethernet on both ends** — Wi-Fi jitter is what will actually ruin this, not average latency.

---

## 3. Risks

| Risk | Severity | Mitigation |
|---|---|---|
| libxenon RF path untested on our specific board revision | **High** | Spike 1 below. This is the single go/no-go item. |
| lwIP built without BSD sockets | Medium | Use the raw UDP API from the start. |
| Big/little-endian mismatch corrupting axes | Medium | Explicit LE wire format + byte-swap helpers + a decode unit test with golden bytes. |
| ViGEmBus archived | Medium | Works today and is signed; hide behind `IVirtualPadSink`. Watch `VirtualPad`. |
| Anti-cheat flags virtual pads | Low–Medium | Rare (DS4Windows is ubiquitous) but not zero — some kernel anti-cheat is hostile to virtual input. Document it; do not design around it. |
| Console must be booted into homebrew | Low | Inherent. Ship a one-touch launcher and document it. |
| Writing our own signed KMDF driver | **Avoided** | Using ViGEmBus removes an EV cert + WHQL attestation dependency entirely. |

---

## 4. Milestones

**Spike 1 — go/no-go (≈1 day).** Build a stock libxenon hello-world `.xex`, run it, and dump `get_controller_data()` for a *wireless* pad to the XeLL serial/telnet console. If button presses show up, the project is de-risked and everything after it is ordinary engineering. If they do not, stop and reassess before writing anything else.

**Spike 2 — transport (≈1 day).** `network_init()` + raw lwIP UDP; blast a fixed test datagram at a PC and confirm receipt with a 20-line C# listener.

**M1 — one pad, buttons only.** Console → PC → ViGEm virtual pad, verified in the Windows "Set up USB game controllers" applet.

**M2 — full state.** Sticks, triggers, correct endianness, deadzone passthrough (raw values — let the game apply its own deadzone).

**M3 — 4 pads + hotplug.** Connect/disconnect mirroring, keepalive timeout.

**M4 — rumble return path.** `FeedbackReceived` → UDP → `usbctrl_set_rumble()`.

**M5 — packaging.** WiX MSI that chains the ViGEmBus installer, plus a tray UI showing per-slot link state, packet loss and RTT. Console side ships as a `.xex` with copy-to-USB instructions.

---

## 5. Rejected alternatives

| Option | Why not |
|---|---|
| Buy a wireless receiver dongle | Correct answer for anyone who just wants to play. Explicitly out of scope. |
| Desolder the RF module and wire it to USB | Works ([well documented](https://agarmash.com/posts/xbox-360-controller-receiver/)) but destroys the console and is "additional hardware". |
| Write our own KMDF/UMDF HID driver | Needs an EV cert + WHQL attestation signing. ViGEmBus already solved this and is signed. |
| Leaked Xbox 360 XDK for the console app | Legally unacceptable. libxenon is clean-room and sufficient. |
| Bluetooth / generic 2.4 GHz dongle | Does not work. The 360 wireless protocol is proprietary and requires Microsoft's RF silicon. |

---

## 6. Bottom line

The risky-sounding parts — the proprietary radio protocol and the Windows kernel driver — are both already solved by existing open source. What we actually have to build is a UDP bridge with careful endianness handling, hotplug semantics, and an installer. **Run Spike 1 before committing further effort**; it converts the one remaining unknown into a yes or a no in about a day.

## References

- [libxenon (Free60Project)](https://github.com/Free60Project/libxenon) — `drivers/usb/usbctrl.c`, `drivers/input/input.c`, `drivers/network/network.h`
- [ViGEmBus](https://github.com/nefarius/ViGEmBus) · [ViGEm.NET](https://github.com/nefarius/ViGEm.NET) · [Nefarius.ViGEm.Client on NuGet](https://www.nuget.org/packages/Nefarius.ViGEm.Client/)
- [VDX — XInput/x360ce to ViGEm sample app](https://github.com/nefarius/VDX)
- [Building a wireless receiver from a broken Xbox 360](https://agarmash.com/posts/xbox-360-controller-receiver/) — RF module pinout and USB IDs
- [Xbox 360 homebrew apps list — ConsoleMods Wiki](https://consolemods.org/wiki/Xbox_360:Homebrew_Apps_List)
