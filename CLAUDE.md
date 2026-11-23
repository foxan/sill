# Sill

Open-source, free app that streams individual Mac app windows to iPhone and iPad
(designed for iPhone Duo first), with a free Mac companion. Tip jar, no
subscription, no servers. Read `docs/BRIEF.md` before doing product-level work.

Formerly winstream; the folder still carries the old name.

The long per-branch progress log, the full file-by-file layout and the full
list of build and test commands that used to live here are in
`docs/claude-md-archive.md`. Each feature's plan and results are in its
`docs/*-plan.md`. Read those when you work on that feature; keep this file to
rules.

## Current work

Status per feature lives in its plan doc and its PR, not here. Do not append
progress logs to this file.

## Layout

- `Package.swift` — SwiftPM. Products: `StreamProtocol` (shared wire format,
  iOS + macOS), `SillHost` (the CLI, target `SillHostCLI`) and `SillMenuBar`
  (Sill.app's executable). `SillHostCore` (the host, folder `Sources/SillHost`)
  is a library target with no product, so the iOS project never sees it.
- `Sources/StreamProtocol/` — the wire format shared by both sides, mainly
  `StreamMessage.swift` (14-byte header + payload framing, message kinds, HEVC
  parameter sets). Change it in one place. Payload types sit beside it
  (Switcher, HostSettings, Remote, Compatibility, MacMenu, Pointer, Gesture,
  Clipboard, ...); the archive lists each.
- `iOSClient/Sill.xcodeproj` — the iPhone/iPad app.
- `Scripts/`, `Tests/` — build, release and test rigs; `site/` — the website.

## Build and run

```
swift build -c release
swift run -c release SillHost                # nothing streams until the iOS app picks a window
swift run -c release SillHost --synthetic    # test pattern; no Screen Recording needed
Tests/checks/run-all.sh                      # every pure check, as CI runs them (~5 min)
Scripts/make-app.sh                          # .build/Sill.app, signed with the Apple Development identity
Scripts/make-app.sh --install --open         # Noah: replace /Applications/Sill.app and launch it
Scripts/release-ios.sh                       # iOS archive and export; uploads nothing
python3 -m http.server 8000 --directory site # the website
```

Needs Xcode as the active developer directory with its license accepted; with
Command Line Tools only, add `--build-system native`. First run prompts for
Screen Recording (the CLI's belongs to Terminal, Sill.app's to Sill). Sill.app
logs to `~/Library/Logs/Sill/Sill.log` (`tail -F`); its settings are
`defaults read me.saffer.sill.mac`. Every other command, flag and `SILL_TEST_*`
variable is in the archive's "Build and run". Test-only rigs that use the
hardware encoder, Accessibility or Screen Recording say so in their names;
don't run those unprompted.

## Compatibility floor

The first public builds (Sill for iPhone and iPad from the App Store, Sill.app
from GitHub) set it for good. Every later host keeps serving devices from the
first public build on, and every later device keeps working with Macs from it,
or each says why (docs/update-notice-plan.md). The exact frozen list is in the
archive's "Compatibility floor"; the rules are:
- Kept as they are: the TLS home door with pairing; the ALPNs `sill/1` and
  `sill-pair/1` (a later `sill/2` only ever goes beside them); the 14-byte
  header; kinds 0–28 and 32 and their payloads; the ping echo; the pairing TXT
  keys `r` and `p`; a `sill-pair/1` connection is exactly one kind 19 then one
  kind 20 (a later method or proof format comes with a later `v`, never a new
  method at `v` 1); pairing is never refused for the device's age.
- Additive only: new fields optional, never renamed or retyped; kind numbers
  never reused (29–31 are taken: audio, PR #66); no new case in an enum an older
  peer decodes; `StreamSource` keeps its three cases. A new input event, scroll
  phase, window command, gesture or clipboard generation goes only to a host
  that said it takes it.
- A device is refused, never served wrong: a host that can't serve older
  devices raises `DeviceGate.minimumDeviceVersion` and they get kind 22
  "update". The host still completes the `sill/1` handshake and reads the
  hello before refusing; never stop offering `sill/1`.
- Every device says hello first (kind 23); every host's window list gives its
  version and protocol. A newer device facing an older Mac says "Update Sill on
  ‹Mac›".
- `SillProtocol.current` (1) rises only with a change an older peer cannot
  skip, and the floor rises with it.
- The hello goes first inside TLS at both doors; the device gate runs once, in
  `Door`. Builds from before home pairing get a failed handshake, never kind 22.

## Conventions

- Swift, Apple frameworks only. No third-party dependencies unless a decision
  in `docs/BRIEF.md` says so. Zero operating costs is a hard rule.
- Threading in the spike is by hand: capture queue → VT callback thread →
  network queue. Keep it explicit and commented rather than reaching for actors
  until the design settles.
- HEVC, no B-frames, real-time mode. AV1 is out (no hardware encode on Apple
  silicon as far as we know).
- Latency beats quality. Drop frames before queuing them.
- Don't add features the current milestone doesn't need.

## Decisions already made (don't relitigate without asking Noah)

- Free app, tip jar, open source. Not paid, not subscription.
- iCloud auto-pairing (same Apple Account, Mac just appears) for milestone 5.
  Bonjour only for the spike.
- Native feel is the bar: Flighty-level polish, iOS conventions, Duo layouts.
- License: Apache-2.0, LICENSE since PR #15, kept when the repository went
  public on 2026-09-26 (paid plan is gone, so no GPL/CLA needed).
- v1 out of scope: hole punching, multi-window, layout customization, audio.
- Home pairing ships before 1.0 (Noah, 2026-09-25): the floor is the TLS home
  door with pairing; no 1.0 device speaks plain TCP to Sill.app.
