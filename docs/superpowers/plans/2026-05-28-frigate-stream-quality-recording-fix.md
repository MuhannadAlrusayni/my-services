# Frigate Stream Quality & Recording Fix Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Fix Frigate live view quality, reduce recording delay, and stop go2rtc RTSP i/o timeout errors across all cameras.

**Architecture:** Three config-only changes — switch go2rtc to TCP RTSP transport (fixes timeouts), remove unnecessary audio pipelines from detect-only sub-streams (reduces load), point each camera's live view at the main stream (fixes quality), and increase the in-memory recording cache from 1Gi to 3Gi (stops segment drop warnings).

**Tech Stack:** Kubernetes, Kustomize, Frigate NVR (v0.16), go2rtc

---

## Files

| File | Change |
|---|---|
| `clusters/production/frigate/frigate-config.yaml` | Rewrite to current multi-camera config with all three fixes applied |
| `clusters/production/frigate/deployment.yaml` | Cache emptyDir sizeLimit 1Gi → 3Gi |

---

### Task 1: Rewrite frigate-config.yaml with all three fixes

The repo file is out of date (single-camera). This task syncs it to the current multi-camera config and applies all fixes in one shot.

**Files:**
- Modify: `clusters/production/frigate/frigate-config.yaml`

- [ ] **Step 1: Replace the entire file content**

Write `clusters/production/frigate/frigate-config.yaml` with this exact content:

```yaml
tls:
  enabled: false

auth:
  cookie_secure: true

mqtt:
  enabled: false

record:
  enabled: true
  retain:
    days: 3
    mode: motion
  alerts:
    retain:
      days: 10
  detections:
    retain:
      days: 10

ffmpeg:
  output_args:
    record: preset-record-generic-audio-aac

audio:
  enabled: true

snapshots:
  enabled: true
  retain:
    default: 10

go2rtc:
  streams:
    sara_room_cam_main:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.173:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
      - ffmpeg:sara_room_cam_main#audio=aac
    sara_room_cam_sub:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.173:554/cam/realmonitor?channel=1&subtype=1&transport=tcp
    entrance_area_cam_main:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.172:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
      - ffmpeg:entrance_area_cam_main#audio=aac
    entrance_area_cam_sub:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.172:554/cam/realmonitor?channel=1&subtype=1&transport=tcp
    living_room_cam_main:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.171:554/cam/realmonitor?channel=1&subtype=0&transport=tcp
      - ffmpeg:living_room_cam_main#audio=aac
    living_room_cam_sub:
      - rtsp://{FRIGATE_CAM_RTSP_USERNAME}:{FRIGATE_CAM_RTSP_PASSWORD}@192.168.100.171:554/cam/realmonitor?channel=1&subtype=1&transport=tcp

cameras:
  sara_room_camera:
    enabled: true
    audio:
      enabled: true
    onvif:
      host: 192.168.100.173
      port: 80
      user: "{FRIGATE_CAM_RTSP_USERNAME}"
      password: "{FRIGATE_CAM_RTSP_PASSWORD}"
      tls_insecure: false
      ignore_time_mismatch: false
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/sara_room_cam_main
          roles:
            - record
            - audio
        - path: rtsp://127.0.0.1:8554/sara_room_cam_sub
          roles:
            - detect
    live:
      stream_name: sara_room_cam_main
    detect:
      width: 704
      height: 480
      fps: 5
  entrance_area_camera:
    enabled: true
    audio:
      enabled: true
    onvif:
      host: 192.168.100.172
      port: 80
      user: "{FRIGATE_CAM_RTSP_USERNAME}"
      password: "{FRIGATE_CAM_RTSP_PASSWORD}"
      tls_insecure: false
      ignore_time_mismatch: false
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/entrance_area_cam_main
          roles:
            - record
            - audio
        - path: rtsp://127.0.0.1:8554/entrance_area_cam_sub
          roles:
            - detect
    live:
      stream_name: entrance_area_cam_main
    detect:
      width: 704
      height: 480
      fps: 5
  living_room_camera:
    enabled: true
    audio:
      enabled: true
    onvif:
      host: 192.168.100.171
      port: 80
      user: "{FRIGATE_CAM_RTSP_USERNAME}"
      password: "{FRIGATE_CAM_RTSP_PASSWORD}"
      tls_insecure: false
      ignore_time_mismatch: false
    ffmpeg:
      inputs:
        - path: rtsp://127.0.0.1:8554/living_room_cam_main
          roles:
            - record
            - audio
        - path: rtsp://127.0.0.1:8554/living_room_cam_sub
          roles:
            - detect
    live:
      stream_name: living_room_cam_main
    detect:
      width: 704
      height: 480
      fps: 5

version: 0.16-0
detect:
  enabled: true
motion:
  enabled: true
```

Key changes vs. old config:
- All 6 `rtsp://` source URLs in `go2rtc.streams` have `&transport=tcp` appended
- `ffmpeg:*_sub#audio=aac` lines removed from all 3 sub-stream entries (sub-streams are detect-only)
- `live: stream_name: <camera>_main` added under each camera block

- [ ] **Step 2: Commit**

```bash
git add clusters/production/frigate/frigate-config.yaml
git commit -m "fix: frigate go2rtc tcp transport, remove sub-stream audio, set live to main stream"
```

---

### Task 2: Increase recording cache in deployment.yaml

**Files:**
- Modify: `clusters/production/frigate/deployment.yaml:77`

- [ ] **Step 1: Change cache emptyDir sizeLimit**

In `clusters/production/frigate/deployment.yaml`, find the `cache` volume (around line 75) and change `sizeLimit`:

```yaml
        - name: cache
          emptyDir:
            medium: Memory
            sizeLimit: 3Gi   # was 1Gi — 4 cameras need ~400MB buffer each
```

- [ ] **Step 2: Commit**

```bash
git add clusters/production/frigate/deployment.yaml
git commit -m "fix: increase frigate recording cache from 1Gi to 3Gi for 4-camera setup"
```

---

### Task 3: Deploy to cluster

The deployment is managed by FluxCD — push to git and it reconciles automatically. The Frigate config file is an exception: it lives on a PVC (not managed by kustomize/FluxCD), so it must be copied to the pod manually. Do the copy first so the new config is on the PVC before FluxCD restarts the pod.

- [ ] **Step 1: Copy updated config to the running pod (updates the PVC)**

```bash
FRIGATE_POD=$(kubectl get pod -n frigate -l app=frigate -o jsonpath='{.items[0].metadata.name}')
kubectl cp clusters/production/frigate/frigate-config.yaml frigate/${FRIGATE_POD}:/config/config.yaml
```

Expected: no output (silent success)

- [ ] **Step 2: Push to git (FluxCD picks up the deployment.yaml change and restarts the pod)**

```bash
git push
```

- [ ] **Step 3: Wait for FluxCD to reconcile**

```bash
flux reconcile kustomization flux-system --with-source
```

Then wait for the pod to be ready:

```bash
kubectl rollout status deployment/frigate -n frigate --timeout=120s
```

Expected:
```
deployment "frigate" successfully rolled out
```

---

### Task 4: Verify

- [ ] **Step 1: Check go2rtc logs — no new i/o timeout warnings**

Open the Frigate UI → Go2rtc tab. Wait 5 minutes. No new `i/o timeout` lines should appear. (A few at startup during initial connect are acceptable — what you're watching for is the pattern of timeouts every 2-5 minutes that was happening before.)

- [ ] **Step 2: Check Frigate logs — no segment drop warnings**

```bash
kubectl logs -n frigate -l app=frigate --since=5m | grep -E "unable to keep up|recording cache"
```

Expected: no output

- [ ] **Step 3: Check live view quality**

Open Frigate UI → click each camera. The live view should now show high-quality main stream video, matching the quality of the direct camera source (compare to Image #3 from the issue).

- [ ] **Step 4: Test recording latency**

Wave in front of a camera to trigger motion detection. In the Frigate UI → Recordings tab, the clip should appear within ~5 seconds. (Previously 10-15s.)

- [ ] **Step 5: Monitor for 10 minutes**

Keep the Frigate UI open and watch for:
- No error spikes in the logs tab
- All 3 cameras showing stable live view without freezing or reconnecting
- go2rtc tab showing no new timeout warnings
