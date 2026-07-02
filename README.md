# Go2 Chrome Console

**A browser-based control console for the Unitree Go2 robot dog, connected over the internet via WebRTC.**

Built from scratch by reverse-engineering the encrypted official mobile app and reimplementing its entire control stack as a web application. No SDK, no same-LAN requirement, no Tailscale. Just open Chrome, connect to the robot from anywhere.

> This repository is a project showcase. The full source code is maintained in a private repository.  
> I'm happy to do a live demo or code walkthrough on request.

---

## What it does

A single-page web console that connects to a Unitree Go2 EDU over the cloud (WebRTC), streams its camera and audio live, drives it with keyboard controls, triggers its full trick and gait repertoire, decodes and renders its LiDAR point cloud in 3D, and runs the robot's own SLAM mapping with autonomous patrol. All of this was reverse-engineered from the encrypted phone app.

---

## The Breakthrough

The core technical challenge was getting a browser to control the robot remotely (different network) over WebRTC.

**The problem:** Every existing open-source project either works only on the same LAN, or uses Python's `aiortc` library which fails the DTLS handshake over Unitree's TURN relay. The connection negotiates ICE to `completed`, then dies during DTLS and times out. This is a known `aiortc` limitation (GitHub issues #593, #413) because its pure-Python DTLS path is fragile over relayed connections.

**The insight:** Unitree's cloud signaling is completely decoupled from the WebRTC engine. Connecting requires just two HTTPS calls: one for TURN credentials, one to exchange the SDP offer/answer wrapped in an AES/RSA envelope. The robot does not care what engine produced the offer.

**The solution:** Let Chrome's built-in `libwebrtc` (the same battle-tested engine the phone app uses) be the WebRTC peer. Python handles only the cloud authentication and SDP envelope. The browser does the actual DTLS handshake, TURN relay, and media.

No public project had done browser-over-cloud control of the Go2 before this. I verified this against every existing open-source Go2 WebRTC project.

```mermaid
flowchart LR
    subgraph Browser["Chrome Tab"]
        B1["WebRTC Engine\n(libwebrtc)"]
        B2["SDP Offer\nDTLS / TURN / Media"]
    end

    subgraph Bridge["Python Bridge"]
        P1["Cloud Login\nTURN Credentials"]
        P2["AES/RSA\nSDP Envelope"]
    end

    subgraph Cloud["Unitree Cloud"]
        C1["Routes by\nSerial Number"]
        C2["TURN Relay\nServer"]
    end

    subgraph Robot["Go2 Robot"]
        R1["ROS 2 / DDS\nOnboard"]
        R2["DTLS Peer\nData Channel"]
    end

    B1 -- "SDP offer" --> P1
    P2 -- "Encrypted SDP" --> C1
    C1 -- "SDP relay" --> R1
    R2 -- "SDP answer" --> C2
    C2 -- "Robot answer" --> P2
    P1 -- "SDP answer" --> B2

    B2 <== "WebRTC Media + Data Channel\n(direct, via TURN relay)" ==> R2

    style Browser fill:#1a1f35,stroke:#3b82f6,color:#e6edf3
    style Bridge fill:#1a2e1a,stroke:#22c55e,color:#e6edf3
    style Cloud fill:#2a1a2e,stroke:#a855f7,color:#e6edf3
    style Robot fill:#2e2a1a,stroke:#f59e0b,color:#e6edf3
```

### The exact fixes that unlocked it (in order)

1. **Cached the login token** so it doesn't re-authenticate per request (re-logging triggered Unitree's API rate limit, HTTP 567).
2. **Chrome, not aiortc, creates the RTCPeerConnection + offer + handles DTLS.** This was the key architectural decision.
3. **Wrapped the SDP in the dog's JSON envelope** before AES-encrypting. Sending a bare SDP returned `code=500`.
4. **Camera-only offer** on the dashboard (drop audio from the initial negotiation) so the dog accepts cleanly.
5. **Validation handshake**: the dog sends a challenge over the data channel; the client replies with the correct hash, then publishes `"on"` to start media.

---

## How the Dog Understands Commands

Once the WebRTC connection is up, there is one data channel named `"data"`. Everything after connect is JSON messages over that single channel, published directly to the robot's internal ROS 2 topics.

The robot is effectively a ROS 2 robot with a WebRTC bridge in front of it. Once you hold that data channel, you have the same control surface the phone app has. The dog does not know or care whether the command came from the official app or this console.

```mermaid
flowchart TD
    subgraph Console["Browser Console"]
        CMD["Command\n(e.g. Dance, Move, Map)"]
    end

    subgraph DataChannel["WebRTC Data Channel"]
        MSG["JSON Message\ntype + topic + data"]
    end

    subgraph Dog["Go2 Onboard ROS 2"]
        SPORT["rt/api/sport/request\nMove, Tricks, Gaits, Flips"]
        SLAM["rt/uslam/client_command\nMap, Localize, Navigate, Patrol"]
        LIDAR["rt/utlidar/switch\nLiDAR On/Off"]
        AUDIO["Audio Stream\nTwo-way Mic/Speaker"]
        TELEM["rt/lf/lowstate\nBattery, Temps, Mode"]
    end

    CMD --> MSG
    MSG --> SPORT
    MSG --> SLAM
    MSG --> LIDAR
    MSG --> AUDIO
    TELEM --> MSG

    style Console fill:#1a1f35,stroke:#3b82f6,color:#e6edf3
    style DataChannel fill:#1a2e1a,stroke:#22c55e,color:#e6edf3
    style Dog fill:#2e2a1a,stroke:#f59e0b,color:#e6edf3
```

---

## What Was Reverse-Engineered

The Unitree app frontend is a Vue/Vite web bundle, AES-encrypted, served by a local NanoHTTPD server inside the APK. I cracked the asset encryption, found the AES key and IV in the app's Java source, decrypted all the JS bundles, and mined the exact wire protocols from them.

What I extracted:
- The full sport command envelope and api-id enums, including the undocumented `mcf`-mode IDs that differ from the SDK documentation
- The uSLAM command vocabulary (plain slash-strings, not JSON) for mapping, localization, navigation, and patrol
- The LiDAR voxel format: LZ4-compressed 128x128xN occupancy bitfield. Reimplemented the decoder from scratch in pure JavaScript.

---

## Features (all working)

| Feature | Description |
|---|---|
| **Live Camera** | WebRTC video stream from the robot's front camera |
| **Two-Way Audio** | Listen to the robot's microphone, talk through its speaker via push-to-talk |
| **WASD Driving** | Keyboard teleop via the robot's sport Move velocity command with speed control |
| **Full Trick Set** | Postures, dances, greetings, gaits (StaticWalk, TrotRun, etc.), AI walks, handstand, flips |
| **LiDAR 3D Viewer** | Decodes the robot's voxel stream into a point cloud, rendered with orbit/zoom/pan camera and live heading arrow |
| **SLAM Mapping** | Build a map using the robot's onboard SLAM, save it, reload it |
| **Autonomous Patrol** | Drop waypoints on the 3D map, run a looping patrol with pause/resume |
| **Live Telemetry** | Battery SOC, voltage, current, motor temps, BMS/IMU temps, mode, gait |
| **Emergency Stop** | SPACE key halts drive, patrol, and navigation instantly |
| **Gesture Control** | Prototype: browser MediaPipe hand detection triggers tricks on the robot |
| **AI Voice Agent** | Prototype: Gemini-powered voice agent using the robot's mic/speaker |

---

## Screenshots

### Full Dashboard: Camera + LiDAR + Controls + Telemetry
![Full dashboard showing live camera feed, 3D LiDAR point cloud, posture and trick controls, drive controls, and live telemetry bar](screenshots/01_dashboard_full.png)

### Dense LiDAR Point Cloud with Full Control Panel
![Dense LiDAR 3D point cloud with the Go2 model visible, control panel showing gaits, AI walks, advanced maneuvers, and flips](screenshots/02_lidar_controls.png)

### SLAM Patrol Mode with Waypoints
![SLAM patrol interface with waypoints placed on the 3D map, patrol Start/Pause/Stop controls, obstacle avoidance toggle, and audio panel](screenshots/03_slam_patrol.png)

### SLAM Mapping Workflow
![SLAM panel showing the full mapping workflow with Start Map, End Map, Save Map buttons, localize in saved map, waypoint placement, and patrol steps](screenshots/04_slam_mapping.png)

### 106K Point LiDAR Scan with Voice Panel
![Densest LiDAR scan with 106,941 points, Go2 3D model centered in the room scan, voice/audio controls and TTS panel visible](screenshots/05_lidar_dense.png)

---

## Demo Video

The full demo video (camera, driving, tricks, LiDAR, SLAM, patrol) is too large for GitHub.

**Watch it here:** *[YouTube link coming soon]*

---

## Autonomy Architecture: Project ROSE

Beyond remote control, I designed a six-layer autonomy architecture for an autonomous gallery docent use case. Each layer is an independent module, built and verified separately, coordinated by a top-level state machine.

```mermaid
block-beta
    columns 1

    block:L6:1
        columns 2
        L6T["L6"] L6C["Orchestrator: PATROL > GREET > CONVERSE > RETURN_BASE > IDLE"]
    end

    block:L5L4:1
        columns 2
        block:left:1
            L5T["L5"] L5C["Self-Care Supervisor\nBattery + temp watchdog\nForces return-to-base"]
        end
        block:right:1
            L4T["L4"] L4C["Voice Agent\nGemini Live API\nPer-gallery knowledge"]
        end
    end

    block:L3:1
        columns 2
        L3T["L3"] L3C["Perception: MediaPipe gesture recognition + visitor detection"]
    end

    block:L2:1
        columns 2
        L2T["L2"] L2C["Patrol Manager: waypoint list, navigation goals, pause/resume"]
    end

    block:L1:1
        columns 2
        L1T["L1"] L1C["Map + Waypoints: one-time SLAM walk, named locations"]
    end

    block:BUS:1
        columns 1
        BUSTEXT["Control Bus: signaling_bridge.py + WebRTC data channel (already built)"]
    end

    style L6 fill:#1f1215,stroke:#ef4444,color:#f87171
    style L5L4 fill:#161b22,stroke:#30363d,color:#e6edf3
    style left fill:#1f1a12,stroke:#f59e0b,color:#fbbf24
    style right fill:#1a1225,stroke:#a855f7,color:#c084fc
    style L3 fill:#121f15,stroke:#22c55e,color:#4ade80
    style L2 fill:#12171f,stroke:#3b82f6,color:#60a5fa
    style L1 fill:#16181d,stroke:#6b7280,color:#9ca3af
    style BUS fill:#1a2332,stroke:#3b82f6,color:#58a6ff
```

**Priority:** Self-Care > Visitor Interaction > Patrol

The key architectural decision: let the robot navigate itself with Unitree's onboard SLAM. The orchestrator stays the high-level brain and never runs a real-time motion loop over the cloud. This keeps the entire system inside the latency envelope.

---

## Tech Stack

| Layer | Technologies |
|---|---|
| **Backend** | Python, aiohttp (async HTTP server), Unitree cloud crypto (AES-CBC + RSA) |
| **Frontend** | Single-page HTML/JS, Three.js (3D LiDAR), WebAudio (two-way audio) |
| **Networking** | WebRTC (Chrome libwebrtc), DTLS, TURN, ICE, data channels |
| **Robotics** | ROS 2 / DDS topics over WebRTC, SLAM, sport/gait control, LiDAR voxel decode |
| **Reverse Engineering** | APK decompilation, AES asset decryption, protocol mining from obfuscated JS/DEX |
| **Perception** | MediaPipe gesture detection, LZ4 decompression (WASM), voxel-to-point-cloud decoder |
| **AI** | Gemini Live API for voice agent (prototype) |

---

## How I Built This

This project was built through a combination of hands-on reverse engineering and AI-assisted development with Claude. The reverse engineering (APK cracking, protocol mining, hardware debugging) was manual work. The software implementation was pair-programmed with Claude, where I directed the architecture and Claude helped write and iterate on the code.

The hardest problems were not code problems. They were:
- Figuring out that `aiortc` fundamentally cannot complete DTLS over a TURN relay, and that the solution was to let Chrome be the WebRTC engine
- Cracking the app's AES-encrypted JS bundles to recover the undocumented command protocol
- Diagnosing a coordinate-frame mismatch (raw lidar-odom vs saved-map frame) that caused every navigation goal to return NO_PATH
- Reimplementing the LiDAR voxel-bitfield decoder from the app's obfuscated source

---

## Prior Art

These projects exist in the Go2 WebRTC space. None of them do browser-over-cloud.

- [legion1581/unitree_webrtc_connect](https://github.com/legion1581/unitree_webrtc_connect) - Python/aiortc, cloud signaling crypto (I reuse this for the crypto layer)
- [tfoldi/go2-webrtc](https://github.com/tfoldi/go2-webrtc) - Browser-based, LAN only
- [phospho-app/go2_webrtc_connect](https://github.com/nicholaswma/go2_webrtc_connect) - Python, cloud
- [lesh/go2-webrtc-deno](https://github.com/lesh/go2-webrtc-deno) - Deno, cloud

---

## License

This repository contains documentation, screenshots, and architectural diagrams only. The source code is proprietary and maintained in a private repository.

Documentation: [CC BY-NC-ND 4.0](https://creativecommons.org/licenses/by-nc-nd/4.0/)

Copyright 2026. All rights reserved.
