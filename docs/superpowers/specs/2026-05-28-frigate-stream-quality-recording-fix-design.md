# Frigate Stream Quality & Recording Fix

**Date:** 2026-05-28  
**Status:** Approved

## Problem

Three symptoms, two root causes:

1. **Live view bad quality** — Frigate's live camera view uses the sub-stream (detect-only, low resolution) instead of the main stream.
2. **Recording delay (10–15s)** — The in-memory recording cache (1Gi) overflows when 4 cameras record simultaneously, causing `record.maintainer` to discard segments.
3. **go2rtc RTSP i/o timeouts** — All cameras (192.168.100.171/172/173) fire repeated `read tcp ... i/o timeout` errors, triggering stream reconnections that degrade quality and interrupt segment writes.

The go2rtc timeout is the primary root cause for issues 2 and 3.

## Approach

Approach B (comprehensive): fix RTSP transport, remove unnecessary pipelines, point live view at main stream, increase cache.

## Changes

### `clusters/production/frigate/frigate-config.yaml`

**1. TCP transport on all RTSP source URLs**

Append `?transport=tcp` to every `rtsp://` URL in `go2rtc.streams`. UDP is unreliable for LAN cameras and causes the i/o timeouts seen in logs.

Applies to all 6 go2rtc streams: `sara_room_cam_main`, `sara_room_cam_sub`, `entrance_area_cam_main`, `entrance_area_cam_sub`, `living_room_cam_main`, `living_room_cam_sub`.

**2. Remove sub-stream audio pipelines**

Delete the `- ffmpeg:*_sub#audio=aac` line from every sub-stream entry. Sub-streams serve the `detect` role only — audio transcoding there adds CPU load and go2rtc pressure with no benefit. Main streams keep their `ffmpeg:*_main#audio=aac` lines since they serve `record`.

**3. Live view → main stream per camera**

Add to each camera block:
```yaml
live:
  stream_name: <camera>_main
```
This makes the Frigate UI display the high-quality main stream instead of the detect sub-stream.

### `clusters/production/frigate/deployment.yaml`

**4. Increase recording cache**

Change the `cache` emptyDir from `sizeLimit: 1Gi` to `sizeLimit: 3Gi`. With 4 cameras each buffering ~400MB of segments, 1Gi overflows and triggers segment discard warnings.

## Files Changed

| File | Change |
|---|---|
| `clusters/production/frigate/frigate-config.yaml` | TCP transport, remove sub audio pipelines, add live.stream_name |
| `clusters/production/frigate/deployment.yaml` | Cache emptyDir 1Gi → 3Gi |

## Test Plan

After deploying:

| Step | What to check | Pass condition |
|---|---|---|
| 1 | go2rtc logs (Go2rtc tab in Frigate UI) | No new `i/o timeout` warnings for 5+ min |
| 2 | Frigate logs | No "unable to keep up with recording segments" warnings |
| 3 | Open live view for each camera | High-quality image (matches direct camera source) |
| 4 | Trigger motion, wait for recording | Recording appears in timeline within ~5s (was 10–15s) |
| 5 | Monitor for 10 min | No error spikes, all streams stable |
