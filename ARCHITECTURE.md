# Personal Media Streaming System: Architecture & Implementation Plan

A comprehensive engineering specification for building a self-hosted personal media streaming ecosystem (Netflix/Plex style) in a **unified monorepo** using **Java 25 LTS** with **Spring Boot 4.1.x** on the server side, a **shared TypeScript Core SDK (`packages/core-sdk`)**, five client platforms (**React 19 Web**, **iOS**, **Android**, **Android TV**, and **Apple TV / tvOS** via React Native / `react-native-tvos`), and a **hybrid LAN/WAN networking model** for secure internet streaming with automatic local network acceleration.

---

## 1. System Architecture & Multi-Client Dual-Connection Topology

The system allows Web, Mobile, and Living-Room TV clients to connect securely from anywhere over the internet while automatically detecting and switching to high-bandwidth direct LAN connections when on home Wi-Fi/Ethernet.

```
+---------------------------------------------------------------------------------------+
|                                    Home PC Server                                     |
|                                                                                       |
|   +-----------------------+     +-----------------------+     +--------------------+  |
|   | Local Media Storage   |---->| Java NIO WatchService |---->| FFprobe Metadata   |  |
|   | (Movies & Shows)      |     | (Directory Ingestion) |     | Parser (Jackson)   |  |
|   +-----------------------+     +-----------------------+     +---------+----------+  |
|                                                                         |             |
|   +---------------------------------------------------------------------v---------+   |
|   |                   Spring Boot 4.1.x Application (Java 25 LTS)                 |   |
|   |  - Java 25 Virtual Threads & Structured Concurrency (`StructuredTaskScope`)   |   |
|   |  - Spring Security 7.x (JWT + Signed Stream Tokens + TV QR/PIN Device Auth)   |   |
|   |  - Spring Data JPA (Jakarta EE 11 + SQLite / PostgreSQL with WAL Mode)        |   |
|   |  - REST Controllers (Catalog, Client Capability Profiles, Heartbeat, VTT)     |   |
|   |  - mDNS Announcer (JmDNS `_locaflix._tcp.local.` for Mobile & TV discovery)   |   |
|   +-------------------------------------------------------------------------------+   |
|                                         |                                             |
|   +-------------------------------------v-----------------------------------------+   |
|   |                 3-Tier Streaming, Remux & Transcoding Engine                  |   |
|   |  1. Direct Play: Spring ResourceRegion (HTTP 206 / 4MB Chunks / 0% CPU)       |   |
|   |  2. Direct Stream (Remux): FFmpeg `-c:v copy` to fMP4 HLS (<2% CPU)           |   |
|   |  3. Full HLS Transcode: GPU HW Accel (NVENC/QSV/AMF/VT) + Keyframe Seek `-ss` |   |
|   +-------------------------------------------------------------------------------+   |
+-----------------------------------|---------------------------------------------------+
                                    |
            +-----------------------+-----------------------+
            |                                               |
            v                                               v
  [ Local Wi-Fi / Ethernet (LAN) ]                [ Secure Internet (WAN) ]
  - Direct IP / mDNS (`192.168.1.x:8080`)         - Cloudflare Tunnel / Caddy HTTPS
  - Zero-latency, 100+ Mbps direct stream         - Let's Encrypt TLS & WAF Protection
  - Uncompressed 4K HDR Direct Play / Remux       - Adaptive Bitrate HLS Transcoding
            |                                               |
            +-----------------------+-----------------------+
                                    |
                                    v
        +-------------------------------------------------------+
        |       Shared TypeScript Core SDK (packages/core-sdk)  |
        |  - Smart LAN/WAN + mDNS Resolver                      |
        |  - Client Capability Profiles & Zustand/TanStack      |
        +---------------------------+---------------------------+
                                    |
        +---------------------------+---------------------------+
        |                           |                           |
        v                           v                           v
+---------------+         +-------------------+       +-------------------+
| Web Client    |         | Mobile Clients    |       | Living-Room TV    |
| (apps/web)    |         | (apps/mobile)     |       | (apps/tv)         |
| - React 19    |         | - React Native    |       | - RN tvOS         |
| - Video.js /  |         | - iOS (AVPlayer)  |       | - Android TV &    |
|   Hls.js      |         | - Android (Media3)|       |   Apple TV (tvOS) |
| - Keyboard/   |         | - Touch, PiP &    |       | - 10-ft D-Pad UI  |
|   Scrub VTT   |         |   Bonjour mDNS    |       | - QR/PIN Pairing  |
+---------------+         +-------------------+       +-------------------+
```

---

## 2. Server Architecture: Java 25 LTS & Spring Boot 4.1.x

### 2.1 Technology Stack & Modern Java 25 Features

| Layer | Technology | Details & Purpose |
| :--- | :--- | :--- |
| **Java Platform** | **Java 25 (LTS)** | Production-grade Virtual Threads, Structured Concurrency (`StructuredTaskScope`), Scoped Values, and pattern-matching enhancements. |
| **Framework** | **Spring Boot 4.1.x (Spring Framework 7.x)** | Jakarta EE 11 baseline, native Virtual Thread concurrency dispatcher, enhanced HTTP Streaming, and GraalVM Native Image readiness. |
| **Security** | **Spring Security 7.x + jjwt** | Stateless JWT authentication, short-lived signed stream tokens (`?streamToken=...`), 6-digit PIN / QR device pairing for Apple TV & Android TV, and dual-origin CORS + Private Network Access (`Access-Control-Allow-Private-Network: true`). |
| **Persistence** | **Spring Data JPA + SQLite / PostgreSQL** | Jakarta Persistence 3.2; SQLite in WAL mode (`journal_mode=WAL`, `synchronous=NORMAL`, `busy_timeout=5000`) with serialized write pooling to prevent `SQLITE_BUSY` under concurrent Virtual Thread workloads. |
| **Media Prober** | **FFprobe CLI + Jackson 3.x** | Scans video/audio streams, HDR/Dolby Vision metadata, container tags, and embedded subtitle tracks into immutable Java 25 Record DTOs. |
| **Streaming Engine** | **Spring ResourceRegion & FFmpeg** | 3-tier engine: `ResourceRegion` Direct Play, `-c:v copy` fMP4 HLS Direct Stream (remux), and GPU-accelerated HLS transcoding with `-ss` keyframe seeking. |
| **Local Discovery** | **JmDNS** | Broadcasts `_locaflix._tcp.local.` on LAN so iOS, Android, Android TV, and Apple TV clients discover the server automatically with zero manual IP setup. |

### 2.2 Per-Platform Streaming Matrix (Direct Play vs. Direct Stream Remux vs. HLS Transcode)

Different client platforms have vastly different native hardware decoders. Every client sends its platform identifier (`X-Locaflix-Client-Platform: WEB | IOS | ANDROID | ANDROID_TV | APPLE_TV`), allowing the server's `StreamingDecisionService` to pick the lowest-CPU, highest-quality path:

| Client Platform | Native Player Engine | Direct Play (`HTTP 206 ResourceRegion`, 0% CPU) | Direct Stream / Remux (`-c:v copy` fMP4 HLS, <2% CPU) | Full HLS Transcode (GPU HW Accel) |
| :--- | :--- | :--- | :--- | :--- |
| **Web (`apps/web`)** | HTML5 `<video>`, `Hls.js` | `MP4`/`WebM` + `H.264`/`AV1`/`VP9` + `AAC`/`Opus`/`MP3` | `MKV` containing `H.264` video -> remuxed to HLS + `AAC` audio | `HEVC` (where unsupported), `DTS`/`TrueHD`, or WAN bitrate limit |
| **iOS (`apps/mobile`)** | Apple `AVPlayer` | `MP4`/`MOV` + `H.264`/`HEVC (HDR10/DV)` + `AAC`/`AC3`/`E-AC3` | `MKV` containing `H.264`/`HEVC` -> remuxed (`-c:v copy`) to fMP4 HLS | `AV1`/`VP9` in incompatible containers, `DTS`/`TrueHD` audio, or WAN cellular |
| **Apple TV (`apps/tv`)** | tvOS `AVPlayer` | `MP4`/`MOV` + `H.264`/`HEVC 4K HDR10/DV` + `AAC`/`AC3`/`E-AC3` | `MKV` containing `H.264`/`HEVC` -> remuxed (`-c:v copy`) to fMP4 HLS | Unsupported video codecs or WAN bandwidth ceiling |
| **Android (`apps/mobile`)** | `Media3 / ExoPlayer` | `MP4`/`MKV`/`WebM` + `H.264`/`HEVC`/`AV1`/`VP9` + `AAC`/`AC3`/`E-AC3`/`Opus`/`FLAC` | Rare (ExoPlayer natively demuxes `.mkv` via HTTP 206 byte ranges) | `TrueHD`/`DTS-HD` on phones without passthrough, or WAN cellular |
| **Android TV (`apps/tv`)** | `Media3 / ExoPlayer` | **Nearly all formats**: `MP4`/`MKV`/`WebM` + `4K HEVC/AV1/H.264` + HDMI eARC passthrough (`TrueHD`/`DTS`/`Atmos`) | Rare (Direct Plays almost everything on LAN) | Only when remote on WAN |

### 2.3 Spring Boot 4.1.x REST API Specification

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/api/v1/auth/login` | Authenticates user; returns JWT access token, refresh token, and profile info. |
| `POST` | `/api/v1/auth/device-code` | Initiates TV 10-foot pairing; returns 6-digit `userCode`, `deviceCode`, and QR payload. |
| `POST` | `/api/v1/auth/device-activate` | Authenticated Web/Mobile client approves a TV's 6-digit `userCode`. |
| `POST` | `/api/v1/auth/device-poll` | TV client polls with `deviceCode`; receives JWT and server LAN/WAN config once approved. |
| `POST` | `/api/v1/auth/stream-token` | Issues a short-lived signed stream token for `<video>`, `AVPlayer`, and `ExoPlayer` URLs. |
| `GET` | `/api/v1/health` | Sub-millisecond ping endpoint for client LAN vs. WAN network detection. |
| `GET` | `/api/v1/library/sections` | Fetches categorized rows (`Featured Hero`, `Continue Watching`, `Movies`, `Series`). |
| `GET` | `/api/v1/media/{id}` | Detailed media record including per-platform `StreamingDecision` (`DIRECT_PLAY`, `REMUX`, `TRANSCODE`). |
| `GET` | `/api/v1/media/{id}/stream` | Direct Play endpoint returning `206 Partial Content` via 4 MB `ResourceRegion` chunks. |
| `GET` | `/api/v1/media/{id}/hls/master.m3u8` | Initiates or attaches to a Direct Stream (fMP4 remux) or adaptive HLS transcode stream. |
| `GET` | `/api/v1/media/{id}/hls/{segment}` | Serves `.ts` or `.m4s` segment (triggering `-ss` keyframe seek if needed). |
| `GET` | `/api/v1/media/{id}/subtitles/{track}` | Serves extracted subtitle tracks in standard WebVTT format. |
| `GET` | `/api/v1/media/{id}/storyboard.vtt` | Serves scrubber timeline preview thumbnails. |
| `POST` | `/api/v1/playback/progress` | Receives playback position heartbeat `{ mediaId, positionSeconds, durationSeconds }`. |

---

## 3. Unified Client Architecture (`packages/core-sdk` + `apps/web`, `apps/mobile`, `apps/tv`)

### 3.1 Monorepo Client Technology Stack

| Workspace | Target Platforms | Tech Stack |
| :--- | :--- | :--- |
| **`packages/core-sdk`** | All 5 Platforms | TypeScript, Zustand, TanStack Query, `connectionManager.ts` (HTTP probe + mDNS/Bonjour hook), Client Capability Profiles, WebVTT Storyboard Parser. |
| **`apps/web`** | Desktop & Mobile Browsers | React 19, Vite, Tailwind CSS, `video.js` / `hls.js`, Lucide React, Radix UI. |
| **`apps/mobile`** | **iOS** & **Android** | React Native (TypeScript), NativeWind, `react-native-video` (AVPlayer / ExoPlayer), Picture-in-Picture (PiP), `react-native-zeroconf` (Bonjour/NSD mDNS discovery), QR Scanner (to pair TVs). |
| **`apps/tv`** | **Android TV** & **Apple TV (tvOS)** | React Native TV (`react-native-tvos`), 10-foot Spatial D-Pad / Siri Remote focus engine (`TVFocusGuideView`, `hasTVPreferredFocus`), QR/PIN Pairing Screen, `react-native-video` with 4K HDR & surround audio passthrough. |

### 3.2 Smart LAN/WAN & mDNS Connection Manager (`packages/core-sdk`)

1. **Zero-Config mDNS Discovery (Mobile & TV)**:
   - On iOS/Apple TV (Bonjour) and Android/Android TV (NSD), the client automatically resolves `_locaflix._tcp.local.` to discover the home server's current LAN IP (`http://192.168.1.x:8080`) without user input.
2. **Fast Health Probe & Failover (All Clients)**:
   - Probes `${localAddress}/api/v1/health` with a `1200ms` timeout.
   - If reachable, routes API calls and video streams over **LAN** (enabling uncompressed 4K Direct Play / Remux).
   - If unreachable, seamlessly switches to `${remoteAddress}` over **HTTPS WAN** (enabling adaptive bitrate HLS).

---

## 4. Secure Internet Streaming (WAN Access)

```
+--------------------------------------------------------------------+
|                   Secure Remote Access Architecture                 |
+--------------------------------------------------------------------+

  [ Web / iOS / Android / Android TV / Apple TV ] (Remote Wi-Fi / 5G)
          |
          v HTTPS (Port 443)
  [ Cloudflare Edge / Let's Encrypt TLS ]
          |
          v Outbound Encrypted Tunnel (Zero Open Inbound Ports)
  +------------------------------------------------------------------+
  |                          Home PC                                 |
  |  - Cloudflare Tunnel daemon (`cloudflared`)                      |
  |    or Caddy Reverse Proxy with Dynamic DNS                       |
  |                        |                                         |
  |                        v localhost:8080                          |
  |        [ Spring Boot 4.1.x Server (Java 25 LTS) ]               |
  +------------------------------------------------------------------+
```

---

## 5. Comprehensive 4-Tier Testing Architecture

| Level | Server (`server/`) | Shared SDK & Clients (`packages/core-sdk`, `apps/web`, `apps/mobile`, `apps/tv`) | Scope & Key Assertions |
| :--- | :--- | :--- | :--- |
| **1. Unit Tests** | **JUnit 5 + AssertJ + Mockito** | **Vitest** | Per-platform `StreamingDecision` (`WEB`, `IOS`, `ANDROID`, `ANDROID_TV`, `APPLE_TV`), FFprobe JSON parsing, byte-range math, JWT/stream-token/device-code crypto, and `core-sdk` LAN/WAN failover. |
| **2. Component-Level Tests** | **`@WebMvcTest` & `@DataJpaTest`** | **Vitest + React Testing Library / RNTL** | Controller slices (`MediaStreamController`, `AuthController` TV device pairing), JPA queries, Web components (`HeroBanner`, `VideoPlayerModal`), Mobile touch player controls, and TV 10-foot focusable cards & PIN pairing screen. |
| **3. Integration Tests** | **`@SpringBootTest` + MockMvc** | **Vitest + MSW (Mock Service Worker)** | SQLite (WAL) concurrency under Virtual Threads, `HTTP 206` `ResourceRegion` streaming, `StructuredTaskScope` FFmpeg remux/transcode pipelines, TV device-code activation workflow, and CORS/PNA filters. |
| **4. End-to-End (E2E) Tests** | **Playwright + Maestro** | **Playwright (Web) + Maestro/Detox (Mobile & TV)** | Full-stack journeys: login & TV QR/PIN pairing, catalog browsing, LAN/WAN switching, Direct Play (`HTTP 206`) & HLS playback, subtitle switching, and cross-device "Continue Watching" resume. |

---

## 6. Step-by-Step Implementation Roadmap

### Milestone 1: Spring Boot 4.1.x Core, Multi-Platform Streaming Engine & SQLite WAL (Weeks 1–2)
- [ ] Initialize Monorepo root and `server/` (Spring Boot 4.1.x on Java 25 LTS with Virtual Threads enabled).
- [ ] Create `User`, `Series`, `Season`, `MediaItem`, `SubtitleTrack`, `PlaybackProgress`, and `DevicePairingCode` entities in SQLite WAL mode.
- [ ] Implement `FolderScannerService` and `FFprobeService` (Java 25 records + `StructuredTaskScope`).
- [ ] Build `StreamingDecisionService` supporting `WEB`, `IOS`, `ANDROID`, `ANDROID_TV`, and `APPLE_TV` capability profiles.
- [ ] Build `MediaStreamController` with 4 MB `ResourceRegion` HTTP 206 Direct Play streaming.
- [ ] Write **Unit, Component (`@WebMvcTest`, `@DataJpaTest`), and Integration (`@SpringBootTest`) tests**.

### Milestone 2: Shared Core SDK (`packages/core-sdk`) & React 19 Web Client (`apps/web`) (Weeks 3–4)
- [ ] Initialize `packages/core-sdk` with TypeScript DTOs, client capability profiles, `connectionManager.ts`, Zustand stores, and TanStack Query hooks.
- [ ] Build `apps/web` (React 19 + Vite + Tailwind CSS): Hero Spotlight, "Continue Watching" carousel, Media Detail modal, and `Video.js`/`Hls.js` player with WebVTT storyboard scrubbing.
- [ ] Write **Unit, Component, Integration, and Playwright E2E tests** for `core-sdk` and `apps/web`.

### Milestone 3: FFmpeg Direct Stream (Remux) & GPU HLS Transcoding Pipeline (Weeks 5–6)
- [ ] Build `TranscodingService` supporting both **Direct Stream Remux (`-c:v copy` to fMP4 HLS)** for iOS/Apple TV and **Full GPU HLS Transcoding** with `-ss` keyframe seeking under Java 25 `StructuredTaskScope`.
- [ ] Add WebVTT subtitle extraction and timeline storyboard sprite generation.
- [ ] Write **Unit, Component, Integration, and E2E tests** for remuxing, HLS seeking, and subtitle delivery.

### Milestone 4: Mobile (`apps/mobile`) & Living-Room TV (`apps/tv`) Clients + Security Hardening (Weeks 7–8)
- [ ] Implement Spring Security 7.x JWT auth, signed `streamToken` URLs, TV 6-digit PIN / QR device pairing endpoints, and JmDNS (`_locaflix._tcp.local.`) broadcaster.
- [ ] Build `apps/mobile` (React Native for **iOS & Android**) with native `AVPlayer`/`ExoPlayer` Direct Play, PiP, mDNS discovery, and TV QR code scanner.
- [ ] Build `apps/tv` (`react-native-tvos` for **Android TV & Apple TV**) with 10-foot D-pad/Siri Remote spatial focus navigation, QR/PIN login screen, and 4K HDR playback.
- [ ] Complete **full-suite Unit, Component, Integration, and E2E tests** across Server, Core SDK, Web, Mobile, and TV workspaces.
