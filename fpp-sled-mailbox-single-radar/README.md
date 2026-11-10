# fpp-sled-mailbox: single-radar patch

**Problem:** with one HLK-LD2410B on `/dev/ttyUSB0`, the plugin never counted anything.
`sled_daemon.py` records a car only when radar A and radar B fire in sequence
(A→B or B→A within `direction_window_s`). With no radar B, `tB` is always empty,
so radar A detections were thrown away.

**What the patch does** (against `focusedonsound/fpp-sled-mailbox` @ `7986aa4`):

- Adds `ld2410.mode` = `auto` (default) | `dual` | `single`.
  - `auto` uses single-radar mode when only one radar port is set, **or** when
    only one of the two configured ports exists at startup. Your current config
    (B = `/dev/ttyUSB1`, which isn't plugged in) switches automatically. No
    config change is needed.
  - In single mode, each new detection counts as one car after `car.cooldown_s`
    (override with `ld2410.single_cooldown_s`). Parked suppression still applies.
    Counts are recorded under the Inbound label (no direction with one radar).
- Masks bit 7 on status, energy and distance high bytes in radar data frames.
  CH340 adapters at 256000 baud flip that bit, which pushed energy above 100 and
  silently dropped the frame.
- Settings page: "None (single radar)" option for Sensor B, plus a Radar Mode selector.
- Reads the parked timeout from `car.parked_timeout_s` (the value the UI saves).

## Apply on the Pi

```bash
cd /home/fpp/media/plugins/fpp-sled-mailbox
curl -fsSL -o /tmp/single-radar.patch https://raw.githubusercontent.com/bobreese/bobreese/claude/loving-curie-und7cm/fpp-sled-mailbox-single-radar/single-radar.patch
git apply --check /tmp/single-radar.patch && git apply /tmp/single-radar.patch
sudo systemctl restart sled-mailbox
grep -E 'Radar mode|falling back|\[Car\]' /home/fpp/media/logs/plugin-fpp-sled-mailbox.log | tail
```

You should see `Radar mode: SINGLE (radar A ...)`, then `[Car] Inbound total=…` lines
as people or cars pass.

To undo: `git checkout -- . && sudo systemctl restart sled-mailbox`.
A plugin update from the FPP Plugin Manager may also overwrite the patch.
