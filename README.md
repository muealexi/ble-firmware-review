# ble-firmware-review

A review pass for BLE devices running Arduino-framework firmware, and for the app that speaks to
them. It looks for the small set of defects that actually reach hardware:

| Tag | What it catches |
|---|---|
| `nvs:` | a path that erases or wears the flash holding enrolled keys |
| `slot:` | a connection slot taken and not returned on an error path |
| `replay:` | a nonce that can repeat, a challenge accepted twice |
| `dfu:` | an update that can leave a partial image running |
| `pairing:` | a secret on the air outside the window meant to gate it |
| `mutex:` | a stack's own structures touched outside its poll loop |
| `budget:` | a wait with no bound, or one outliving what it guards |
| `key:` | key material reaching a log, an error string, or the wrong storage |

It reports one line per finding and never edits. Style, naming, architecture and test coverage
belong to other passes.

## Scope

Arduino-framework firmware — `arduino-cli`, NimBLE-Arduino, HomeSpan. **ESP-IDF and FreeRTOS are
deliberately out of scope**: their patterns differ enough that advice written for one misleads in
the other. An ESP-IDF sibling would be a second skill, not a widening of this one.

The pass reviews **both ends of the protocol**. A replay window enforced in firmware and ignored
by the client is a defect in the client, and only a reviewer reading both can see the mismatch.

## Install

```
/plugin marketplace add muealexi/ble-firmware-review
/plugin install ble-firmware-review@ble-firmware-review
```

## Adding a check

A check enters the list only when someone can name the defect it would have caught. Checks
admitted for being generally good practice fire on everything, which teaches readers to skim the
output and takes the useful checks down with them.
