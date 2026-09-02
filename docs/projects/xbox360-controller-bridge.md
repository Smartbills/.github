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

Only ~3–6 ms of that is ours to win — see §3 for what is reducible and what is floor.

---

## 3. Reducing latency

Split the budget into what we control and what we do not. Most of the total is not ours.

### 3.1 The floor we cannot move

- **RF link cadence, ~4–8 ms.** Microsoft's radio decides how often a pad reports. No amount of code changes it.
- **The game's own XInput poll: ~16.7 ms at 60 fps, ~6.9 ms at 144 fps.** Most engines poll input once per frame, so the game quantises our input to its frame cadence regardless of how fast we deliver it.

Those two terms dominate. Of the ~8–15 ms the bridge adds over a direct dongle, only roughly **3–6 ms is actually ours to win**. The realistic goal is *indistinguishable from a dongle*, not *faster than a dongle* — and once we are inside one frame of a dongle, further optimisation is invisible to the player.

### 3.2 Console side — the biggest single win

**Never log in the hot loop.** At 115200 baud 8N1 a character costs ~87 µs, so one 40-character `printf` costs ~3.5 ms — larger than every other term we control, combined. This is the classic homebrew latency killer. Gate all logging behind a build flag, or ring-buffer it and drain it inside the keepalive tick.

Beyond that:

- Keep the loop tight and unconditional: `usb_do_poll()` → read pads → send changed → `network_poll()`. No sleeps, no frame sync, no video init, no allocation.
- **Do not batch the four pads.** Emit each pad's datagram the instant its state differs; waiting to coalesce adds a full poll interval for no bandwidth benefit we need.
- **Investigate the interrupt endpoint interval** libxenon queues for the RF module. Its OHCI stack is full-speed, so the frame floor is 1 ms; if the endpoint descriptor is queued at a 4 or 8 ms interval, dropping it to 1–2 ms is a direct win — bounded, of course, by how often the radio actually has new data.

### 3.3 PC side

- **Dedicated foreground thread with a blocking `ReceiveFrom`, `ThreadPriority.Highest`.** Do *not* use `async`/`await` in the receive path — thread-pool scheduling adds hundreds of microseconds and occasional whole milliseconds under load. A blocking receive on its own thread is both simpler and faster here.
- **Zero allocation in the hot path.** `Span<byte>`, `stackalloc`, pre-allocated buffers; no LINQ, no boxing, no closures. Set `GCSettings.LatencyMode = GCLatencyMode.SustainedLowLatency`. A gen2 pause is tens of milliseconds and dwarfs everything else on this list — the fix is to never allocate, not to tune the collector.
- **Drain to newest.** If more than one datagram is queued for a pad, decode only the highest sequence number and discard the rest. This is precisely what the sequence field is for: after any hiccup, stale frames must be dropped rather than played back, or lag accumulates instead of recovering. Keep `ReceiveBufferSize` modest so a stall cannot bank a large backlog.
- **Process priority High** — not Realtime, which can starve the very input stack we depend on.
- **`SubmitReport()` event-driven on change only.** Never run a fixed-rate submission loop; it adds a quantisation step we would then have to fight.
- If anything in the loop ever sleeps or waits on a timer, call `timeBeginPeriod(1)`. Windows' default timer resolution is 15.6 ms and is per-process since Windows 10 2004. A blocking receive avoids needing this at all.

### 3.4 Network and NIC

Bandwidth is a non-issue — a 62-byte frame serialises in ~5 µs on the 360's 100 Mbit link. Optimise queueing and jitter, never throughput.

- **Wired only.** Wi-Fi and powerline introduce jitter, and jitter is what is actually felt.
- **Ideally a direct link:** 360 straight into a second PC NIC with static IPs, no switch. Removes contention and bufferbloat from other LAN traffic.
- **Disable interrupt moderation / Interrupt Throttle Rate** on the PC NIC (set to off or the lowest setting). Default moderation deliberately banks packets for 50–200 µs.
- **Disable Energy Efficient Ethernet / Green Ethernet.** On a low-rate, mostly-idle stream the link keeps entering low-power idle and paying a wake cost on every packet. The 100 ms keepalive helps hold it awake too.

### 3.5 Explicitly do not do these

- **No input prediction or extrapolation.** Overshoot on stick movement feels far worse than the latency it hides.
- **No smoothing, filtering or interpolation of axis values.** Pass raw values through and let the game apply its own deadzone and response curve.
- **No fixed-rate resampling stage** anywhere in the chain. Every such stage adds half its period to average latency.

### 3.6 Measure, do not guess

Track **p50, p99 and max — never the mean.** A 10 ms link with 2 ms of jitter feels better than an 8 ms link that spikes to 15 ms.

- **In-band:** the console stamps the PPC timebase (`mftb()`) into each packet, the agent echoes it back, and the console computes round-trip and halves it. Measuring RTT rather than one-way sidesteps the fact that the two clock domains are unrelated.
- **Ground truth:** 240 fps phone slow-motion of a button press through to on-screen response, with the same test on a real dongle as the control. This is the only measurement that captures every hidden term — radio, game poll and display pipeline included.
- **Ship criterion:** within one frame of the dongle control at the same refresh rate.

### 3.7 Escape hatch

If measured latency is still unacceptable after all of the above, the RF-module-to-USB desolder mod removes the console, the LAN and the bridge entirely: the module attaches to PC USB and Windows' in-box XUSB driver handles it, giving dongle-identical latency. It costs a console and breaks the no-additional-hardware constraint, so it is a fallback, not a plan.

---

## 4. Risks

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

## 5. Milestones

**Spike 1 — go/no-go (≈1 day).** Build a stock libxenon hello-world `.xex`, run it, and dump `get_controller_data()` for a *wireless* pad to the XeLL serial/telnet console. If button presses show up, the project is de-risked and everything after it is ordinary engineering. If they do not, stop and reassess before writing anything else.

**Spike 2 — transport (≈1 day).** `network_init()` + raw lwIP UDP; blast a fixed test datagram at a PC and confirm receipt with a 20-line C# listener.

**M1 — one pad, buttons only.** Console → PC → ViGEm virtual pad, verified in the Windows "Set up USB game controllers" applet.

**M2 — full state.** Sticks, triggers, correct endianness, deadzone passthrough (raw values — let the game apply its own deadzone).

**M3 — 4 pads + hotplug.** Connect/disconnect mirroring, keepalive timeout.

**M4 — rumble return path.** `FeedbackReceived` → UDP → `usbctrl_set_rumble()`.

**M5 — packaging.** WiX MSI that chains the ViGEmBus installer, plus a tray UI showing per-slot link state, packet loss and RTT. Console side ships as a `.xex` with copy-to-USB instructions.

---

## 6. Rejected alternatives

| Option | Why not |
|---|---|
| Buy a wireless receiver dongle | Correct answer for anyone who just wants to play. Explicitly out of scope. |
| Desolder the RF module and wire it to USB | Works ([well documented](https://agarmash.com/posts/xbox-360-controller-receiver/)) but destroys the console and is "additional hardware". |
| Write our own KMDF/UMDF HID driver | Needs an EV cert + WHQL attestation signing. ViGEmBus already solved this and is signed. |
| Leaked Xbox 360 XDK for the console app | Legally unacceptable. libxenon is clean-room and sufficient. |
| Bluetooth / generic 2.4 GHz dongle | Does not work. The 360 wireless protocol is proprietary and requires Microsoft's RF silicon. |

---

## 7. Bottom line

The risky-sounding parts — the proprietary radio protocol and the Windows kernel driver — are both already solved by existing open source. What we actually have to build is a UDP bridge with careful endianness handling, hotplug semantics, and an installer. **Run Spike 1 before committing further effort**; it converts the one remaining unknown into a yes or a no in about a day.

## References

- [libxenon (Free60Project)](https://github.com/Free60Project/libxenon) — `drivers/usb/usbctrl.c`, `drivers/input/input.c`, `drivers/network/network.h`
- [ViGEmBus](https://github.com/nefarius/ViGEmBus) · [ViGEm.NET](https://github.com/nefarius/ViGEm.NET) · [Nefarius.ViGEm.Client on NuGet](https://www.nuget.org/packages/Nefarius.ViGEm.Client/)
- [VDX — XInput/x360ce to ViGEm sample app](https://github.com/nefarius/VDX)
- [Building a wireless receiver from a broken Xbox 360](https://agarmash.com/posts/xbox-360-controller-receiver/) — RF module pinout and USB IDs
- [Xbox 360 homebrew apps list — ConsoleMods Wiki](https://consolemods.org/wiki/Xbox_360:Homebrew_Apps_List)
