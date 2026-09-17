---
name: ble-firmware-review
description: >
  Review a diff for the defects that reach hardware on a BLE device: erased key
  storage, leaked connection slots, replayed nonces, partial firmware images,
  secrets on the air or in a log. Covers Arduino-framework firmware and the app
  that speaks to it, both ends of the protocol. Use when reviewing BLE firmware,
  a pairing, OTA or key-handling path, or the client that drives one. Reports
  one line per finding; never edits.
---

# BLE firmware review

Eight checks. Each one exists because it names a defect that shipped somewhere, and none of
them is a style opinion. If a check ever fires on something that is not a defect, the check is
wrong and should be narrowed — not the finding argued away.

**Scope:** Arduino-framework firmware (`arduino-cli`, NimBLE-Arduino, HomeSpan and friends), and
the client that speaks to it. That framework sits on ESP-IDF, so **the IDF calls the Arduino core
re-exports are in scope and are the point** — `esp_ota_begin`, `esp_ota_set_boot_partition`,
`esp_ota_mark_app_valid_cancel_rollback`, the `nvs_*` family, the partition APIs. The `dfu:` and
`nvs:` checks are largely about exactly those.

Out of scope is the *bare-metal* IDF project shape — `app_main`, components and CMake, menuconfig —
and FreeRTOS task primitives used as the concurrency model: `xTaskCreate`, queues, semaphores. A
sketch's `loop()` and a task scheduler fail differently enough that advice for one misleads in the
other.

**Review both ends.** A replay window enforced in firmware and ignored by the app is a defect in
the app. A length the app sends and the firmware trusts is a defect in the firmware. When only one
end is in the diff, check the invariant against the other end anyway and say so.

## Output

One line per finding. Nothing else — no summary, no preamble, no restating the diff.

`<file>:L<line>: <tag> <what>. <consequence>.`

One line means one line, however wide. A real path spends forty columns before the finding starts,
so these run past eighty; never wrap one to fit a terminal, because a two-line finding is the thing
this format exists to prevent.

Tags: `nvs:` `slot:` `replay:` `dfu:` `pairing:` `mutex:` `budget:` `key:`

❌ "The DFU path might want to verify before it commits — have you considered power loss?"

✅ `d_dfu.ino:L88: dfu: boot partition set before the HMAC. Power loss then boots unverified.`

❌ "Consider whether this connection handling could leak under error conditions."

✅ `g_ble.ino:L142: slot: bad-length return leaves the slot held. Release in the exit path.`

✅ `Opening.swift:L51: replay: nonce compared, never consumed. Two writes on one read both open.`

Report nothing when there is nothing. A review that invents a finding to look thorough is worse
than a silent one, because the next person stops reading the output.

## The checks

### `nvs:` flash and non-volatile storage

Any path that erases, reformats or migrates persistent storage, reached by anything other than an
explicit deliberate action. Debug and serial handlers are the usual culprit: a single keystroke
wired to a wipe erases the partition holding every enrolled key, and the device comes back looking
factory-fresh with no way to tell what happened.

Also: a write in a loop or on every connection. Flash has finite erase cycles, and a counter
persisted on each packet will wear a sector out in the field, not on the bench.

### `slot:` connection slots

A BLE peripheral has a small, fixed number of connection slots. Check every path that takes one
for an early `return`, a `break`, or a thrown error between acquiring and releasing it. One leaked
slot per failed attempt means the device stops accepting connections after a handful of retries,
and the symptom — "it just stopped responding" — looks like a crash rather than a leak.

### `replay:` nonces and challenge freshness

A nonce that can repeat, a challenge accepted twice, a counter that resets on reboot while the
peer's does not, or a window that stays open after the thing it guarded finished. Check that the
value is consumed on use and not merely compared, and that a reconnect cannot rewind it.

**This check is about acceptance, never refusal.** A stale challenge that produces a *denial* — two
taps racing, a write answering a nonce the peer has already rotated — is the freshness working.
It may still be a bug worth someone's time; it is not this one, and tagging it `replay:` teaches
the reader that the tag means "something about nonces" rather than "a door opened that should not
have".

### `dfu:` update atomicity

The question is always the same: can the device end up running a partial image? Check that the
image is verified *before* anything is committed, that an abandoned transfer restarts from the
beginning rather than resuming into a stale buffer, and that losing power midway leaves the
previous firmware intact and bootable.

### `pairing:` window state

A key, a setup code or an enrolment secret put on the air outside the window that was supposed to
gate it. Check that the window's shut state is honoured by every path that can send, not only the
one that opens it, and that a cancel takes effect before the next write rather than after it.

### `mutex:` stack-owned structures

Reading or mutating a protocol stack's own state — a controller list, a connection table, a
characteristic registry — outside the loop or lock the stack owns it under. This reads correctly
in testing and corrupts under a concurrent event, which is the worst failure shape there is: it
passes review, passes the bench, and bites in the field.

### `budget:` waits and timeouts

Any wait without a bound, and any bound that outlives what it guards. A scan with no deadline, a
read that blocks the event loop, a timeout longer than the watchdog, a retry loop with no ceiling.
State the number the bound should have, not merely that one is missing.

### `key:` key material

A key, nonce or setup code that reaches a log, a serial print, an error string, or storage that
is not the secure one. Check error paths especially — the happy path rarely prints secrets, and
the handler written at 2am for a bug frequently does.

## What this skill does not do

Style, naming, architecture, test coverage, and generic code smells belong to other passes; this
one produces nothing about them. It also does not edit — a pass that both finds and fixes protocol
bugs unattended is the last thing anyone wants near a lock.

## Adding a check

A check enters this list only when someone can name the defect it would have caught. Anything
admitted on the grounds that it is generally good practice will fire on everything, teach readers
to skim the output, and take the other seven checks down with it.
