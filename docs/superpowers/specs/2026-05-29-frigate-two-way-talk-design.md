# Frigate Two-Way Talk Implementation Design

**Date:** 2026-05-29  
**Status:** Approved

## Problem

Two-way talk is not enabled on any camera. The existing go2rtc streams all have `#backchannel=0` (added to fix reconnect errors), which prevents go2rtc from establishing the audio output backchannel required for talk.

## Approach

Approach A: Add a `_twoway` go2rtc stream per camera. These streams point to the same main RTSP source but without `#backchannel=0`, so go2rtc can establish the backchannel when talk is active. Frigate's WebRTC viewer activates the microphone button automatically when it finds a `{camera_name}_twoway` stream. The existing streams are untouched — recording, detection, and audio remain unaffected.

## Changes

### `clusters/production/frigate/frigate-config.yaml`

Add three entries to `go2rtc.streams` — one per camera, using the main stream RTSP URL (subtype=0) without `#backchannel=0`:

```yaml
sara_room_camera_twoway:
  - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.173:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
entrance_area_camera_twoway:
  - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.172:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
living_room_camera_twoway:
  - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.171:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
```

No camera-level config changes. No deployment changes. The `_twoway` streams only connect to the camera when two-way talk is actively used — idle cost is zero.

## Files Changed

| File | Change |
|---|---|
| `clusters/production/frigate/frigate-config.yaml` | Add 3 `_twoway` entries to `go2rtc.streams` |

## Test Plan

| Step | What to check | Pass condition |
|---|---|---|
| 1 | Frigate UI → camera live view (WebRTC mode) | Microphone button appears |
| 2 | Click mic, speak | Audio plays on camera speaker |
| 3 | Check Frigate logs after talk session | No errors or reconnects on existing streams |
| 4 | Verify recording still works | Motion clip appears in Recordings tab within ~5s |
