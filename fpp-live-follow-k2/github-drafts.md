# Drafts for pgianotto/fpp-live-follow (not posted yet)

---

## A. Comment on issue #3 (Support native servo capes / co-pwm)

Update: the `co-pwm` approach from this issue is now running and tracking on FPP `10.x-master-3378-g4ec27d33a` with the K2-Pi-Servo cape, on top of fpp-live-follow `8679709`.

What the working patch does in `daemon.py`:

- `_find_pca9685_output()` checks `GET /api/channel/output/co-pwm` first and falls back to `co-other`, so existing PWM-tab setups keep working.
- For `co-pwm` PCA9685 outputs it builds the port list from `outputs` and uses each port's own `startChannel + 1` as the overlay channel (`co-pwm` is 0-based, `PUT /api/overlays/range` is 1-based).
- Center is `center` if present, otherwise `int((min + max) / 2)`, matching `PCA9685Output::loadPortConfig`.
- 0–180° covers the port's calibrated min..max (0 → min, 90 → center, 180 → max). fppd applies `reverse` itself.
- `_set_fpp_pca9685_output()` and `_ensure_fpp_pca9685_enabled()` use whichever file the PCA9685 output was found in, so they no longer touch `co-other`.

In `content.php`, the Pan/Tilt dropdowns read the same output (`outputs` or `ports`). On the cape they now list `Port 1 — DmxSkull - Pan`, `Port 2 — DmxSkull - Tilt` and so on, and the startup log shows `Loaded 16 PCA9685 ports from co-pwm.` instead of `No PCA9685 port configs found; using mock backend.`

Still happy to open a PR with this.

---

## B. New issue

**Title:** Tracking drives the head to its limits when the camera isn't on the head; add camera-mount mode and optional torso servo

### Summary

`LiveTrackingMode._step()` is a closed loop on the head: it keeps turning pan/tilt until the subject is centered in the frame. That only works when the camera turns with the head. If the camera is fixed to the body (or the scene), turning the head never changes the picture. The error never shrinks, so pan and tilt run to their min/max and stay there. Tilt is especially bad, because the subject is rarely vertically centered within the deadzone.

### Environment

- Raspberry Pi 4, FPP `10.x-master-3378-g4ec27d33a`, K2-Pi-Servo cape
- fpp-live-follow `8679709` (core library `b6f63a07`)
- EMEET SmartCam C960 mounted on the animatronic's chest (it turns with the torso, not the head)
- Head pan/tilt servos plus a separate torso rotation servo

### Steps to reproduce

1. Mount the camera on the body rather than the head.
2. Start tracking (`fpp_overlay`, Always On) and stand to one side, slightly above or below camera center.
3. The head turns toward you and keeps going until it hits its limit, even after it's pointing at you. Tilt does the same.

### Proposed change (working locally)

A `camera_mount` setting (`head` = current behavior, the default; `body` = camera fixed to the torso or scene) and a `BodyCamTrackingMode(LiveTrackingMode)` that overrides `_step()`:

- **Torso (optional third servo, `channel_torso`):** closed loop on the horizontal frame error, at up to `servo_torso_speed` °/s. This is the axis that actually moves the camera.
- **Head pan/tilt:** open loop, aimed from the subject's position in the frame: `center + head_aim × 2 × (pos − 0.5)`, rate-limited by each servo's speed. As the torso catches up, the subject drifts to the middle of the frame and the head eases back to center ("head leads, body follows").
- The pan/tilt invert settings apply to the head only. `LiveTrackingMode.update()` flips `sx` for pan invert, so the torso loop undoes that flip. Without that, ticking Pan Invert also reversed the torso and sent it to its end stop.
- `stop_tracking()` and `_handle_overlay_release()` delete the torso overlay along with pan and tilt.
- `/api/status` also reports `torso` and `subject_x`, which made tuning much easier.

For a head-mounted camera with a torso servo, the torso simply follows a share (`torso_follow`) of the head's pan offset.

UI: a Camera Mount select, Head Aim pan/tilt inputs, a Torso column in the servo table, and a Torso Channel select with a None option.

### Separate small UI bug: values unreadable on FPP 10

FPP 10's bootstrap sets `input[type='number'] { color: var(--bs-body-color) }`. Its specificity (0,1,1) beats `.af-input` (0,1,0), so every number box on the plugin page shows near-black text on the `#0f3460` background. Fix:

```css
.af-card input.af-input { color:#e0e0e0 !important; -webkit-text-fill-color:#e0e0e0; }
```

(Don't override the select background as well: that hides FPP's dropdown arrow.)

Happy to open a PR with all of this.
