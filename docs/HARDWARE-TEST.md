# Hardware verification checklist

Run this per model before changing its status from `vendor` to `verified`.
Copy the block, fill in the model and the date, keep it in the pull request.

Use the **Log** tab throughout: every written packet should match what the vendor driver
writes for the same action. A difference is not automatically a bug, but it must be explained.

---

## Template

```markdown
### <Model> — <family> — <date> — <tester>

- [ ] Device appears in the browser picker with the expected name
- [ ] Model is identified correctly (not "unknown")
- [ ] Connection type reported correctly (USB / 2.4 GHz)
- [ ] Firmware version shown
- [ ] Battery percentage shown and plausible
- [ ] Lighting: switching effect changes the keyboard
- [ ] Lighting: brightness +1 → visible change → -1 restores
- [ ] Per-key: paint 3 keys → visible on device → clear
- [ ] Key remap: assign one key → test it → restore default
- [ ] Macro: 3 steps → trigger → delete
- [ ] Setting (debounce/sleep) written → unplug → replug → value persisted
- [ ] No packet errors in the log
- [ ] Writing while the vendor driver is open produces a clear error, not a silent failure

Notes:
```

---

## Per-family specifics

### g5kb (F75 family)
- The six-byte identity must match the table in [PROTOCOL-NOTES.md](PROTOCOL-NOTES.md#1-g5kb--sinowealth-bycombo4-f75-ailesi).
- The LED block is read-modify-write: verify that changing brightness alters **exactly one**
  byte at `0x3A + 2i` and leaves everything else intact.
- Compare the LED block dump before and after: any unexpected changed byte means the
  read-modify-write went wrong.

### aula04 (S75 Pro family)
- Verify the `AA 55` trailer survives on the keymap and per-key blocks.
- For S75 Pro only: the screen upload needs the second HID collection (4096-byte output
  reports). Confirm the app finds it automatically.
- L99: check the settings byte shift (winLock at 4, Alt+F4 at 3, Alt+Tab at 2), not the
  S75 Pro offsets.
- F75 Max: confirm the app reports "sent", never "verified" — there is no read-back command.

### aa (F87 Pro Sonix)
- Every packet must be answered before the next goes out. If a write works only sometimes,
  the echo wait is too short.

### bytetech / xsyd (Hall-effect)
- Read the actuation point back after writing and compare to what was written.
- For xsyd: depth values are micrometres. A value written as `0.10 mm` must read back as
  `100`, not `10`.
- Never send a command outside the five `SAFE` opcodes on `hfd`.

### Mice
- Select each DPI stage on the mouse itself after writing, and confirm the stage colours.
- For `evision`: unknown bytes in the 128-byte config block (`9`, `73`, `75`, `79–127`)
  must be unchanged after a write. Diff the block to prove it.
- Polling rate applies separately to cable and receiver on most mice — test both.

---

## Known failure modes

| Symptom | Likely cause |
| --- | --- |
| Device listed but `open()` fails | Vendor driver still running, or Linux udev rule missing |
| Identity query returns nothing | Keyboard is in Bluetooth mode — configuration is not exposed there |
| Lighting works, keys do not | Wrong model for this VID:PID; check the identity bytes |
| First write works, second does not | Missing acknowledgement wait between packets |
| Settings do not survive replug | `save`/`commit` command not sent after the write |
