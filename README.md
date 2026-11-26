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
| [docs/SCHEMA.md](docs/SCHEMA.md) | Driver contract — how each model describes itself |
| [docs/SUPPORTED.md](docs/SUPPORTED.md) | Model × feature matrix |
| [docs/HARDWARE-TEST.md](docs/HARDWARE-TEST.md) | Manual verification checklist |

---

## Coverage

Roughly 36 models. Each one is its own driver: its own USB IDs, its own identity check,
its own command set. Sharing a VID:PID with another model is common — identity decides,
never the USB ID alone.

| Model | Wired | Receiver | Hardware status | Documented by |
| --- | --- | --- | --- | --- |
| F75 | 258A:010C | 3554:FA09 | verified | open-aula, AulaSofwareLinux |
| F75 Pro | 258A:010C | 3554:FA09 | vendor code | open-aula, AulaSofwareLinux |
| F87 | 258A:010C | 3554:FA09 | vendor code | open-aula, AulaSofwareLinux |
| F87 Pro | 258A:010C | 3554:FA09 | vendor code | open-aula, AulaSofwareLinux |
| F99 | 258A:010C | 3554:FA09 | vendor code | AulaSofwareLinux |
| F99 Pro | 258A:010C | 3554:FA09 | vendor code | AulaSofwareLinux |
| F108 | 258A:010C | 3554:FA09 | vendor code | AulaSofwareLinux |
| H108 | 258A:010C | 3554:FA09 | vendor code | AulaSofwareLinux |
| S75 Pro | 0C45:800A | 0C45:FEFE, 05AC:024F | verified | open-aula |
| L99 | 0C45:800A | 05AC:024F | vendor code | AulaToolkit |
| F108 Pro | 0C45:800A | 05AC:024F | vendor code | parsiya/f108-pro |
| F75 Max | 0C45:800A | 05AC:024F | vendor code (write-only) | FaizAsta/Aula-F75-software |
| F75 HE | 38A6:2729 | 0C45:FEFE | vendor code | FaizAsta/Aula-F75-software |
| F87 Pro (Sonix) | 38A6:271x | 0C45:FEFC | vendor code | open-aula |
| HERO 68 / 84 HE | 372E:103E | 372E:1000, 106C | verified | shizunavn/OpenAula |
| WIN 68 HE MAX | 1CA2:1901 | — | verified | shizunavn/OpenAula |
| SC620 | A8A4:2255 | A8A5:2255 | buttons verified | open-aula |
| SC900 family | 093A / 248A / 379A | same | verified | AulaToolkit |
| SC390 | 32C2:0024 | — | vendor code | AulaToolkit |
| V9 / X9 family | 320F:22B6/22B7/22B8 | 320F:22A0 | vendor code | shDavlatbek/aulaar |

Models without a public protocol are listed but stay read-only, and the app says so.
Models marked *vendor code* show an **Experimental** badge: writing works but has not been
confirmed on physical hardware.

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
work of the open-source community — each file in `protocols/` names the repository, the
licence and the specific source it was adapted from at the top.
