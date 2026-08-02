# Pico de Gallo — server

**Pico de Gallo** is a downstream distribution based on
[Apache Guacamole](https://guacamole.apache.org/) — a fork of
[apache/guacamole-server](https://github.com/apache/guacamole-server) at the
1.6.0 release. This is the guacd proxy: the 1.6.0 release plus multi-monitor
RDP support and a set of stability fixes, developed and tested against a real
deployment. The companion web application is
[pico-de-gallo-client](https://github.com/Slopapalooza/pico-de-gallo-client).

Pico de Gallo is not affiliated with or endorsed by the Apache Software
Foundation. "Apache Guacamole" and "Apache" are trademarks of the ASF; this
distribution retains the Apache-2.0 license and NOTICE and credits upstream
throughout, but is not Apache Guacamole.

Upstream's original build documentation is in the plain [`README`](README)
file.

## What this fork adds

### Multi-monitor support (GUACAMOLE-288)

Backport of upstream draft PR
[apache/guacamole-server#560](https://github.com/apache/guacamole-server/pull/560)
(author: Corentin Soriano) onto the 1.6.0 release tag. Each monitor is a
separate browser window; guacd renders one combined framebuffer and announces
a true per-monitor layout to the RDP server over the Display Update channel.

### Fixes to the multi-monitor code

Found by code review and pilot testing on top of the backport:

- `resize_needed` was read uninitialized in `guac_rdp_disp_alloc()`.
- Closing a middle monitor left stale `x_position`/`left_offset` values in
  the shifted monitors, which were reported verbatim in the RDP monitor
  layout; they are now recomputed after the array shifts.
- Layout slot 0 can be closed while other monitors remain — with
  position-based ordering, the leftmost slot is not necessarily the client's
  main window.
- The `multimon-layout` layer parameter is emitted on **every** layout change
  via a shared `guac_rdp_disp_send_layout()`, not only on desktop resizes. A
  pure monitor reorder keeps the combined dimensions unchanged, so no resize
  fires — clients were left rendering stale slice mappings (content from one
  monitor appearing in another window).
- A full-extent Refresh Rect PDU is sent after each `SendMonitorLayout`, so
  the RDP server repaints regions it does not consider changed after a
  reorder.
- Layout JSON separators are emitted before each element after the first, so
  a skipped (uninitialized) trailing monitor cannot produce unparseable JSON.
- The display engine's rewrite-as-copies optimization pass is disabled: a
  copy's source region may lie in a monitor slice that a cropped client
  window does not have, leaving black rectangles (worst while dragging
  windows between screens). Re-encoding the affected regions is always
  correct, at some bandwidth cost.

### Upstream stability fixes cherry-picked from `main`

Post-1.6.0 fixes from upstream, applied with `git cherry-pick -x` (each
commit records its origin): display-engine hang (#693), ABBA deadlock between
display duplication and frame flush (#689), render-planner infinite loop
(#646), worker-thread race in layer clearing (#640), RDP busy-loop on
transport failure (#635), intermittent RDP error handling (#656), shutdown
lock-ordering crash (#653), missing-`config.h` segfaults (#649), zombie
process accumulation (#617), resource leaks (#685), display-dup fixes (#664,
#657), stream-index reuse delay (#632), and `CMAKE_BUILD_TYPE=Release` for
the FreeRDP Docker build (#660).

### Upstream regression fixed here

The config-file reorganization merged in
[apache/guacamole-server#685](https://github.com/apache/guacamole-server/pull/685)
parses `guacd.conf` even when the file does not exist, reading from an
invalid descriptor — guacd exits in a crash loop (`Bad file descriptor`)
under the default Docker setup, where no `guacd.conf` is mounted. Fixed on
this branch; upstream `main` does not carry the broken structure.

## Building

The stock Dockerfile works as-is, with one caveat: it auto-selects the newest
FreeRDP 2.x tag, which can fail `configure` as a development snapshot. Pin
the version shipped in the official 1.6.0 image:

```bash
docker build --build-arg WITH_FREERDP=2.11.7 -t guacd:multimon .
```

## Requirements and caveats

- Multi-monitor is RDP-only; connections must use
  `resize-method=display-update` and set `secondary-monitors` ≥ 1.
- The RDP host must support the Display Update channel (Windows 8.1 /
  Server 2012 R2 or later).
- Monitors are arranged side by side (per-monitor vertical offsets exist in
  the protocol but are currently locked off in the client — vertical offsets
  produced partially cut-off rendering pending upstream's vertical-reorder
  work).
- This branch tracks a moving upstream draft; expect it to be superseded
  when GUACAMOLE-288 merges.

## License

[Apache License 2.0](LICENSE), same as upstream. The multi-monitor backport
is derived from the upstream draft PR by Corentin Soriano; upstream
cherry-picks retain their original authorship.
