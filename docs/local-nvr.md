# Local NVR: a cloud IP camera made fully self-hosted

I took a generic Wi-Fi IP camera that normally depends on a vendor phone app and cloud, and turned it into a fully local, no-cloud security camera: isolated on the network, cut off from its vendor cloud, recorded to my own NAS, with a live view I can reach from my admin network and a person-detection alert to a chat channel. No camera video or metadata leaves the house.

All addresses, names and identifiers here follow the same sanitizing as the rest of this repo.

## Goals

- Keep the camera's video entirely on-premises. No vendor cloud, no third-party app relay.
- Stop the camera phoning home.
- Record continuously to the NAS with a sensible retention policy.
- Get a live view that works from the admin VLAN, across the firewall, without poking new holes.
- Get a notification when a person appears, not on every bit of motion.

## Network isolation and cutting off the cloud

The camera sits on the isolated **IoT VLAN**, which by design cannot open connections into any other internal zone. On first setup I watched its DNS queries and outbound traffic: it was resolving and connecting to an overseas cloud service to register and stream.

I added a firewall policy that **blocks that camera's outbound internet** entirely (its fixed address to the external zone, all protocols), while leaving it reachable on the LAN. After that:

- The camera can no longer reach its vendor cloud or any internet host. Vendor telemetry and remote cloud viewing stop at the firewall.
- It is still reachable locally for the NVR to pull video.
- Its internet is blocked specifically, so the rest of the IoT VLAN is unaffected.

Blocking its internet also means it can no longer reach a public time server, so its on-screen clock drifts; the NVR's own recording timestamps come from the host clock and stay correct, so this is cosmetic only.

## Local video instead of the vendor app

Rather than the vendor app and cloud, the camera is consumed through its **local RTSP stream**. I probed the camera to find which ports and stream paths it exposes, confirmed the main and sub streams, and pointed the NVR at them directly. The RTSP credentials live only in a restricted-permission environment file on the host and never appear in any config in version control or in this repo.

## Frigate NVR on the hypervisor

[Frigate](https://frigate.video/) runs as a pinned Docker container on the Docker VM on the Proxmox host. It:

- **Records to the NAS over NFS.** A dedicated NFS share is exported to the Docker VM only and mounted on the host; recordings land there, not on the hypervisor's boot SSD. Retention keeps recent footage continuously plus a longer motion-based window, and is easy to change.
- **Keeps its database and config on local disk,** so only the bulky video sits on the network share.
- **Runs object detection** on the CPU (the hypervisor's iGPU is already dedicated to the media container), tracking people.
- **Keeps secrets out of the repo.** The camera password and the notifier's bot token each live in their own `chmod 600` env file on the host; the compose files and detector config reference them by variable only.

## HD live view across VLANs, without new ports

The live view had to work from the **admin VLAN** while the camera and NVR sit on other VLANs, and the first attempt just spun forever.

- The NVR uses **go2rtc** to pull each camera stream **once** and share it, so the camera serves a single connection instead of one per consumer (record, detect and live all read from go2rtc).
- The default live path uses **WebRTC**, which needs a reachable media port and a valid ICE candidate. Across separate VLANs, with no extra port opened, WebRTC could not establish and the player hung.
- The fix was to serve the live view over **MSE** (Media Source Extensions), which rides the same web port the browser already uses, so it traverses the firewall with no new rules. I disabled WebRTC in go2rtc so the player uses MSE directly, and added a transcoded AAC audio track for the live stream (the camera's native audio codec is not MSE-compatible, which was a second reason the player stalled).
- Result: a smooth **HD live view** from the admin workstation, across VLANs, with no WebRTC port opened anywhere.

## Person-detection alert over MQTT

- A small **mosquitto** MQTT broker runs in its own container on a shared Docker network. Frigate publishes events to it.
- A small notifier service subscribes to Frigate's events and, on a **person** detection, posts a push message to a chat channel through a bot. It fires the moment a person first appears, is de-duplicated per event, and timestamps the alert in local time.
- The alert is person-only, so routine motion (shadows, the camera's own clock overlay) does not generate noise.

## Verifying it end to end

I confirmed each stage with live checks rather than assuming:

- Recordings actually appear on the NAS share and grow over time.
- The live view plays from the admin workstation (not a frozen frame: the on-screen clock advances).
- During a controlled walk-through, I watched the MQTT traffic live and traced the whole chain: object detection, the person event, the review alert, the MQTT message, and the notifier's successful post.

## A detection bug I traced live

The first walk-through produced no alert, and the NVR showed zero detections ever. Watching the pipeline live showed the detector was running constantly but finding nothing, and the review page stayed empty, which ruled out the broker and the notifier (they were simply never fed).

The cause was that object detection was running on the camera's **very low-bitrate sub-stream**, where a moving person compresses into a smear that never scores above the detection threshold. Moving detection to the **main (HD) stream** fixed it immediately: the next walk-through produced a person event above threshold, the review alert, the MQTT message and a successful push notification, all confirmed in a live capture.

## Skills shown

VLAN segmentation and device isolation; zone-based firewall policy (blocking one device's outbound internet); Docker and Compose; Frigate NVR; NFS share design and mounting; MQTT (mosquitto); go2rtc restreaming; cross-VLAN live-streaming troubleshooting (WebRTC vs MSE, codec compatibility); event-driven notifications; and secret handling with restricted-permission env files kept out of version control.
