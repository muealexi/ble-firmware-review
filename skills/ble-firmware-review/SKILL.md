---
name: ble-firmware-review
description: >
  Review a diff for the defects that bite BLE devices running Arduino-framework
  firmware: a debug path that erases the flash partition holding the keys, a
  connection slot taken and never returned, a nonce or pairing window that
  outlives what it guards, a firmware update that can leave a partial image
  running, a stack's own structures read outside its poll loop, an unbounded
  wait, a key that reaches a log. Reviews both ends of a protocol — the firmware
  and the app that speaks to it — because the seam between them is where these
  live. Use when reviewing changes to BLE firmware, a pairing or DFU path, or
  the client that drives one. Reports; it does not edit.
---

# BLE firmware review

Eight checks. Each one exists because it names a defect that shipped somewhere, and none of
them is a style opinion. If a check ever fires on something that is not a defect, the check is
wrong and should be narrowed — not the finding argued away.

**Scope:** Arduino-framework firmware (`arduino-cli`, NimBLE-Arduino, HomeSpan and friends), and
the client that speaks to it. ESP-IDF and FreeRTOS APIs are deliberately out of scope: their
patterns differ enough that advice written for one misleads in the other.

**Review both ends.** A replay window enforced in firmware and ignored by the app is a defect in
the app. A length the app sends and the firmware trusts is a defect in the firmware. When only one
end is in the diff, check the invariant against the other end anyway and say so.

## Output

One line per finding. Nothing else — no summary, no preamble, no restating the diff.

`<file>:L<line>: <tag> <what>. <consequence>.`

Tags: `nvs:` `slot:` `replay:` `dfu:` `pairing:` `mutex:` `budget:` `key:`

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
