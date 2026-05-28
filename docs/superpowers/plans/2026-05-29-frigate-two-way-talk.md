# Frigate Two-Way Talk Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Enable two-way talk on all 3 Frigate cameras by adding `_twoway` go2rtc streams that allow backchannel audio without affecting recording or detection.

**Architecture:** Add one `_twoway` go2rtc stream entry per camera in `frigate-config.yaml`, pointing to the same main RTSP source (subtype=0, TCP) but without `#backchannel=0`. Frigate's WebRTC viewer activates the microphone button automatically when it finds a `{camera_name}_twoway` stream. The `_twoway` stream only connects when talk is actively used — zero idle cost.

**Tech Stack:** Kubernetes, FluxCD, Frigate NVR (v0.16), go2rtc

---

## Files

| File | Change |
|---|---|
| `clusters/production/frigate/frigate-config.yaml` | Add 3 `_twoway` entries to `go2rtc.streams` |

---

### Task 1: Add `_twoway` streams to frigate-config.yaml

**Files:**
- Modify: `clusters/production/frigate/frigate-config.yaml`

- [ ] **Step 1: Add the three `_twoway` stream entries**

In `clusters/production/frigate/frigate-config.yaml`, add these entries inside `go2rtc.streams`, after the existing `living_room_cam_sub` entry:

```yaml
    sara_room_camera_twoway:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.173:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
    entrance_area_camera_twoway:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.172:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
    living_room_camera_twoway:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.171:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
```

Note: no `#backchannel=0` on these URLs — that is intentional. The backchannel must be available for two-way talk to work.

The full `go2rtc.streams` block should look like this after the change:

```yaml
go2rtc:
  streams:
    sara_room_camera:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.173:554/cam/realmonitor?channel=1&subtype=0&transport=tcp#backchannel=0
      - ffmpeg:sara_room_camera#audio=aac
    sara_room_cam_sub:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.173:554/cam/realmonitor?channel=1&subtype=1&transport=tcp#backchannel=0
    entrance_area_camera:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.172:554/cam/realmonitor?channel=1&subtype=0&transport=tcp#backchannel=0
      - ffmpeg:entrance_area_camera#audio=aac
    entrance_area_cam_sub:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.172:554/cam/realmonitor?channel=1&subtype=1&transport=tcp#backchannel=0
    living_room_camera:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.171:554/cam/realmonitor?channel=1&subtype=0&transport=tcp#backchannel=0
      - ffmpeg:living_room_camera#audio=aac
    living_room_cam_sub:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.171:554/cam/realmonitor?channel=1&subtype=1&transport=tcp#backchannel=0
    sara_room_camera_twoway:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.173:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
    entrance_area_camera_twoway:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.172:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
    living_room_camera_twoway:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.171:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
```

- [ ] **Step 2: Commit**

```bash
git add clusters/production/frigate/frigate-config.yaml
git commit -m "feat: add two-way talk go2rtc streams for all cameras"
```

---

### Task 2: Deploy to cluster

- [ ] **Step 1: Copy updated config to the running pod**

```bash
FRIGATE_POD=$(kubectl get pod -n frigate -l app=frigate -o jsonpath='{.items[0].metadata.name}')
kubectl cp clusters/production/frigate/frigate-config.yaml frigate/${FRIGATE_POD}:/config/config.yaml
```

Expected: no output (silent success)

- [ ] **Step 2: Push to git**

```bash
git push
```

- [ ] **Step 3: Restart Frigate to pick up the new go2rtc streams**

```bash
kubectl rollout restart deployment/frigate -n frigate
kubectl rollout status deployment/frigate -n frigate --timeout=120s
```

Expected:
```
deployment "frigate" successfully rolled out
```

---

### Task 3: Verify

- [ ] **Step 1: Confirm Frigate started cleanly**

```bash
kubectl logs -n frigate -l app=frigate --since=2m | grep -E "Config Validation|not valid|Extra inputs"
```

Expected: no output (no config errors)

- [ ] **Step 2: Confirm `_twoway` streams appear in go2rtc**

```bash
kubectl exec -n frigate deployment/frigate -- wget -qO- http://localhost:1984/api/streams 2>/dev/null
```

Expected: JSON output containing `sara_room_camera_twoway`, `entrance_area_camera_twoway`, `living_room_camera_twoway`.

- [ ] **Step 3: Check microphone button appears in live view**

Open Frigate UI → click a camera → switch to WebRTC mode. A microphone icon should appear in the player controls.

- [ ] **Step 4: Test two-way talk**

Click the microphone button and speak. Audio should play through the camera's speaker.

- [ ] **Step 5: Confirm existing streams are unaffected**

```bash
kubectl logs -n frigate -l app=frigate --since=5m | grep -E "unable to keep up|recording cache|wrong media"
```

Expected: no output
