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

Arduino-framework firmware — `arduino-cli`, NimBLE-Arduino, HomeSpan — and the client that speaks
to it. That framework sits on ESP-IDF, so **the IDF calls the Arduino core re-exports are in
scope**: `esp_ota_*`, `nvs_*`, the partition APIs. The `dfu:` and `nvs:` checks are mostly about
them. What is out of scope is the bare-metal IDF project shape and FreeRTOS task primitives as the
concurrency model; that would be a second skill, not a widening of this one. The skill states the
scope and the both-ends rule in full.

## Install

```
/plugin marketplace add muealexi/ble-firmware-review
/plugin install ble-firmware-review@ble-firmware-review
```

## Adding a check

The bar, and the reason for it, are in
[`skills/ble-firmware-review/SKILL.md`](skills/ble-firmware-review/SKILL.md) — the file the model
actually reads, and the one place this rule lives.
