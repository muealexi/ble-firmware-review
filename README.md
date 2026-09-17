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
to it, including the ESP-IDF calls the Arduino core re-exports. Bare-metal IDF and FreeRTOS task
primitives are out. The skill draws the line and states the both-ends rule in full.

## Install

```
/plugin marketplace add muealexi/ble-firmware-review
/plugin install ble-firmware-review@ble-firmware-review
```

## Does it work

Backtested blind against two commits from a real BLE opener project, each taken *before* the fix
that followed, reviewed by agents told nothing about what was in them.

| Diff | Findings | Verified against the fix that shipped |
|---|---|---|
| 130 lines of firmware | 3 | 3 |
| 780 lines of Swift, chosen as a control | 2 | 1 |

Five findings, four real, no fabrications. The firmware case found **two defects beyond the one it
was set** — a board falling back to HomeSpan's published `DEFAULT_SETUP_CODE`, and a factory wipe
reaching the HAP namespace from the wrong task — and its remedy for the third, `setSerialInputDisable(true)`,
is what that project shipped. The fifth finding was real behaviour under the wrong tag, and became
the boundary now written into `replay:`.

## Adding a check

The bar, and the reason for it, are in
[`skills/ble-firmware-review/SKILL.md`](skills/ble-firmware-review/SKILL.md) — the file the model
actually reads, and the one place this rule lives.
