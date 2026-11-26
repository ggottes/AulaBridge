# AulaBridge

A browser-based configurator for AULA keyboards and mice. No installer, no background
service — it talks to the device directly over WebHID.

Lighting, per-key RGB, key remapping, macros, DPI stages, sensor settings, screens and a
full packet log, in one page, for every AULA family we can reverse-engineer.

> **Unofficial.** Not affiliated with, endorsed by or supported by AULA. "AULA" and the
> product names are trademarks of their respective owners. Writing a wrong configuration
> to a device is usually recoverable with the vendor tool, but there is no warranty.

---

## Status

Early development. The plan and protocol notes are complete; the application is being built.

| Document | Contents |
| --- | --- |
| [PLAN.md](PLAN.md) | Scope, architecture, UI, security model, phases, risks |
| [docs/PROTOCOL-NOTES.md](docs/PROTOCOL-NOTES.md) | Byte-level protocol reference for all 12 families |
| [docs/SUPPORTED.md](docs/SUPPORTED.md) | Model × feature matrix |
| [docs/HARDWARE-TEST.md](docs/HARDWARE-TEST.md) | Manual verification checklist |

---

## Coverage

Six keyboard protocol families and five mouse families, covering roughly 36 models.

| Family | Models | Hardware status |
| --- | --- | --- |
| `g5kb` | F75, F75 Pro, F87, F87 Pro, F99, F99 Pro, F108, H108 | F75 verified |
| `aula04` | S75 Pro, L99, F108 Pro, F75 Max, F98 Pro, S98 Pro, S99 Pro | S75 Pro verified |
| `aa` | F87 Pro (Sonix variant) | vendor code |
| `hfd` | F75 HE | vendor code |
| `bytetech` | HERO 68/84/75/99/87 HE, WIN68HE Ultra, MINI 60HE | HERO 68/84 verified |
| `xsyd` | WIN 68 HE MAX | verified |
| `mouse55` · `incott` · `yxt` · `evision` · `hubmouse` | SC620, SC900 family, SC390, V9/X9 family, SC800 Max | SC900 Pro verified |

Models marked as vendor-code-only are shown with an **Experimental** badge. Writing to them
is possible but unverified against physical hardware.

### Not supported

- Firmware updates — no recoverable path exists in any public source.
- Headsets and speakers — no HID configuration protocol.
- Bluetooth configuration — those devices expose only standard keyboard/mouse HID.

---

## Requirements

- Desktop Chrome, Edge, Brave or another Chromium-based browser (WebHID).
- The official AULA driver must be closed — only one program can hold the HID interface.
- Linux: a udev rule so the browser can open the device. The app generates it for you.

Firefox, Safari and mobile browsers do not support WebHID.

---

## Development

There is no build step and no dependency. The JavaScript you edit is the JavaScript that runs.

```bash
python -m http.server 8765
```

Open `http://localhost:8765` — WebHID requires HTTPS or `localhost`.

Run the tests:

```bash
node --test
```

---

## License

GPL-3.0-or-later. Protocol knowledge in this project was derived from the reverse-engineering
work of the open-source community — see the source notes at the top of each file in
`protocols/` and in [docs/PROTOCOL-NOTES.md](docs/PROTOCOL-NOTES.md).
