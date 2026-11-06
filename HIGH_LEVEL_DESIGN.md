# Locaflix — High-Level Design (HLD)

## 1. Executive Summary & System Vision

**Locaflix** is a self-hosted personal media streaming ecosystem engineered to deliver a Netflix/Plex-caliber browsing and playback experience across **five client platforms**:
- **Web (`apps/web`)**: Desktop & mobile browsers (React 19 + Vite + Tailwind CSS + Video.js / Hls.js)
- **iOS (`apps/mobile`)**: iPhone & iPad (React Native + Apple `AVPlayer` + Picture-in-Picture)
- **Android (`apps/mobile`)**: Phones & Tablets (React Native + `Media3 / ExoPlayer`)
- **Android TV (`apps/tv`)**: Smart TVs, Shield TV, Chromecast with Google TV (`react-native-tvos` + `ExoPlayer` + 10-foot D-pad UI)
- **Apple TV (`apps/tv`)**: tvOS (`react-native-tvos` + `AVPlayer` + Siri Remote focus navigation)

All five clients share a unified TypeScript core package (**`packages/core-sdk`**) and communicate with a **Java 25 LTS / Spring Boot 4.1.x** backend over a **hybrid LAN/WAN networking topology**.

### 1.1 Core Design Principles
1. **3-Tier Client-Aware Playback (Direct Play -> Direct Stream Remux -> HLS Transcode)**:
   - **Tier 1 — Direct Play (`0%` CPU)**: Serve raw byte ranges (`HTTP 206 Partial Content`, 4 MB `ResourceRegion` chunks) whenever the target device (`WEB`, `IOS`, `ANDROID`, `ANDROID_TV`, `APPLE_TV`) can natively decode the container and codecs (e.g., Android TV `ExoPlayer` playing 4K HEVC MKV directly).
   - **Tier 2 — Direct Stream / Remux (`<2%` CPU)**: When a client (such as iOS or Apple TV `AVPlayer`) supports the video codec (`HEVC`/`H.264`) but not the `.mkv` container, remux on the fly (`ffmpeg -c:v copy`) into fragmented MP4 (`fMP4`) HLS without re-encoding video.
   - **Tier 3 — Adaptive HLS Transcoding (GPU Accelerated)**: Isolate full video/audio transcoding into managed `ffmpeg` subprocesses with `-ss` keyframe seeking when on WAN or when video codecs are unsupported.
2. **Structured Subprocess Isolation**: Supervise all `ffprobe` and `ffmpeg` worker processes using Java 25 Virtual Threads and `StructuredTaskScope` with non-blocking stdout/stderr reading and automatic 60s idle session cleanup.
3. **Zero-Config LAN Discovery & Zero-Trust WAN Access**: Broadcast `_locaflix._tcp.local.` via JmDNS so iOS, Android, Android TV, and Apple TV clients auto-discover the home server on Wi-Fi/Ethernet, and seamlessly fall back to an outbound-only Cloudflare Tunnel / Caddy HTTPS endpoint when away from home.
4. **Form-Factor Native UX**: Deliver a dark-mode cinematic experience tailored to each input paradigm—mouse/keyboard on Web, touch/gestures/PiP on Mobile, and **10-foot spatial D-pad / Siri Remote focus navigation with QR/6-digit PIN device pairing** on Android TV & Apple TV.
5. **Comprehensive 4-Tier Verification**: Enforce automated quality gates across Unit, Component-Level, Integration, and End-to-End (Playwright + Maestro/Detox) test suites.

---

## 2. System Context & Container Architecture

### 2.1 System Context Diagram (Level 1)

```mermaid
flowchart TB
    WebUser["💻 Web User (Desktop/Laptop Browser)"]
    MobileUser["📱 Mobile User (iOS & Android App)"]
    TVUser["📺 Living-Room User (Android TV & Apple TV)"]
    Admin["🛠️ Home Server Owner (Local Media Storage)"]

    subgraph HomeNetwork["Home Network (Gigabit LAN / Wi-Fi)"]
        Locaflix["🎬 Locaflix Server\n(Spring Boot 4.1.x on Java 25 LTS)"]
        FileSystem[("📂 Local Media Storage\n(Movies, Series, Subtitles, Artwork)")]
        mDNS["📡 JmDNS / Bonjour\n(_locaflix._tcp.local.)"]
    end

    subgraph EdgeInfrastructure["External / Edge Infrastructure"]
        Tunnel["🔒 Cloudflare Tunnel / Caddy HTTPS Proxy"]
        TMDB["🌐 Optional TMDB Metadata API"]
    end

    Admin -->|"Drops MKV / MP4 / SRT files"| FileSystem
    FileSystem -->|"Java NIO WatchService"| Locaflix
    Locaflix -->|"Broadcasts LAN IP & Port"| mDNS
    mDNS -.->|"Zero-config discovery"| MobileUser
    mDNS -.->|"Zero-config discovery"| TVUser
    Locaflix -.->|"Fetches optional metadata"| TMDB

    WebUser -->|"LAN Direct Play / WAN HTTPS"| Locaflix
    MobileUser -->|"LAN Direct Play / Remux"| Locaflix
    TVUser -->|"LAN 4K HDR Direct Play / Remux"| Locaflix

    WebUser -->|"Remote WAN HTTPS"| Tunnel
    MobileUser -->|"Remote Cellular 5G HTTPS"| Tunnel
    TVUser -->|"Remote WAN HTTPS"| Tunnel
    Tunnel -->|"Outbound Encrypted Tunnel -> localhost:8080"| Locaflix
```

### 2.2 Monorepo Container & Subsystem Architecture (Level 2)

```mermaid
flowchart TB
    subgraph Clients["Unified Client Apps"]
        WebApp["apps/web\n(React 19 + Vite + Tailwind + Video.js/Hls.js)"]
        MobileApp["apps/mobile\n(React Native iOS & Android + AVPlayer/ExoPlayer)"]
        TvApp["apps/tv\n(React Native tvOS & Android TV + 10-ft D-Pad + QR Auth)"]
    end

    subgraph SharedSDK["packages/core-sdk (Shared TypeScript Package)"]
        ConnMgr["Smart LAN/WAN + mDNS Connection Manager"]
        CapProfiles["Client Capability Profiles\n(WEB, IOS, ANDROID, ANDROID_TV, APPLE_TV)"]
        SharedStores["Zustand Stores & TanStack Query Hooks\n(Auth, Catalog, Playback Heartbeat, Device Pairing)"]
    end

    subgraph ServerApp["server/ (Spring Boot 4.1.x + Java 25 LTS + Virtual Threads)"]
        SecurityFilter["Spring Security 7.x\n(JWT + Stream Tokens + TV Device Code + CORS/PNA)"]
        Controllers["REST API Controllers\n(Auth, DevicePairing, Health, Library, Stream, HLS, Playback)"]
        DecisionSvc["StreamingDecisionService\n(Direct Play vs. fMP4 Remux vs. GPU Transcode)"]
        IngestionSvc["Library Ingestion & FFprobe Service\n(Java NIO WatchService + Jackson 3.x Records)"]
        TranscodeSvc["HLS Remux & Transcoding Supervisor\n(StructuredTaskScope + ProcessBuilder)"]
        JpaLayer["Spring Data JPA Repositories"]
    end

    subgraph HostResources["Host OS & Persistence Layer"]
        SQLite[("SQLite Database\n(WAL Mode, busy_timeout=5000)")]
        MediaDisk[("Media Filesystem\n(.mp4, .mkv, .srt, .vtt)")]
        ScratchDir[("Transcode & Sprite Cache\n(.m3u8, .ts, .m4s, Storyboard .vtt)")]
        FFmpegBin["FFmpeg / FFprobe Binaries\n(NVENC / QSV / AMF / VideoToolbox)"]
    end

    WebApp --> SharedSDK
    MobileApp --> SharedSDK
    TvApp --> SharedSDK

    SharedSDK -->|"REST JSON + X-Locaflix-Client-Platform"| SecurityFilter
    WebApp -->|"HTTP 206 / HLS"| SecurityFilter
    MobileApp -->|"HTTP 206 / fMP4 HLS"| SecurityFilter
    TvApp -->|"HTTP 206 / fMP4 HLS"| SecurityFilter

    SecurityFilter --> Controllers
    Controllers --> DecisionSvc
    Controllers --> JpaLayer
    Controllers --> TranscodeSvc
    IngestionSvc --> MediaDisk
    IngestionSvc --> FFmpegBin
    IngestionSvc --> JpaLayer
    TranscodeSvc --> FFmpegBin
    TranscodeSvc --> ScratchDir
    JpaLayer --> SQLite
```

---

## 3. Core Subsystems & Responsibilities

| Subsystem | Primary Components | Key Responsibilities |
| :--- | :--- | :--- |
| **1. Library Ingestion & Metadata** | `FolderScannerService`, `LibraryWatchService`, `FFprobeService`, `ArtworkService` | Monitors media directories via Java NIO `WatchService`; parses Movie and `SxxExx` Episode filenames; extracts codecs, HDR/Dolby Vision profiles, audio channels, and subtitle streams via `ffprobe`; generates poster thumbnails and WebVTT storyboard sprites. |
| **2. Multi-Client Streaming Engine** | `StreamingDecisionService`, `MediaStreamController`, `HlsController`, `TranscodingService` | Matches media streams against `ClientPlatform` capabilities (`WEB`, `IOS`, `ANDROID`, `ANDROID_TV`, `APPLE_TV`); serves 4 MB `ResourceRegion` (`HTTP 206`) for Direct Play; performs `-c:v copy` fMP4 HLS remuxing for iOS/Apple TV MKVs; runs GPU-accelerated HLS transcoding with `-ss` keyframe seeking. |
| **3. Catalog & Cross-Device Progress** | `LibraryController`, `PlaybackController`, `CatalogService`, `PlaybackService` | Serves categorized sections (`Featured Hero`, `Continue Watching`, `Movies`, `TV Series`, genres); syncs 10-second playback heartbeats across Web, Mobile, and TV so a user can pause on Apple TV and resume at the exact second on iOS or Web. |
| **4. Security, TV Pairing & Networking** | `SecurityConfig`, `JwtTokenProvider`, `StreamTokenService`, `DevicePairingService`, `MdnsService` | Issues stateless JWTs and short-lived `?streamToken=` URLs; manages 6-digit PIN / QR code pairing for Android TV & Apple TV; broadcasts `_locaflix._tcp.local.` via JmDNS; enforces CORS and `Access-Control-Allow-Private-Network: true`. |
| **5. Shared Core SDK (`packages/core-sdk`)** | `connectionManager.ts`, `clientProfiles.ts`, `authStore.ts`, `useCatalog.ts`, `usePlaybackHeartbeat.ts` | Centralizes all TypeScript DTOs, mDNS + HTTP health probing, automatic LAN/WAN failover, token refresh, and playback progress syncing across all 5 client platforms. |
| **6. Platform Client Apps (`web`, `mobile`, `tv`)** | `apps/web`, `apps/mobile`, `apps/tv` | Implements form-factor-optimized UIs: keyboard/mouse & hover scrub thumbnails on Web; touch gestures, PiP, and QR scanner on iOS/Android; 10-foot D-pad/Siri Remote spatial focus navigation and QR login screen on Android TV/Apple TV. |

---

## 4. End-to-End Architectural Flows

### 4.1 TV 10-Foot QR / 6-Digit PIN Device Pairing Flow

```mermaid
sequenceDiagram
    autonumber
    actor User
    participant TV as Apple TV / Android TV (apps/tv)
    participant Server as Spring Boot Server
    participant Mobile as iOS / Android / Web (Logged In)

    TV->>Server: POST /api/v1/auth/device-code { platform: "APPLE_TV", deviceName: "Living Room Apple TV" }
    Server-->>TV: 200 OK { deviceCode: "d8f9...", userCode: "842-910", qrPayload: "locaflix://pair?code=842910", expiresIn: 300 }
    TV->>TV: Renders 6-Digit PIN "842-910" & QR Code on TV Screen

    loop Every 3 Seconds
        TV->>Server: POST /api/v1/auth/device-poll { deviceCode: "d8f9..." }
        Server-->>TV: 202 Accepted { status: "PENDING" }
    end

    User->>Mobile: Scans QR Code or enters "842-910" in Mobile/Web App
    Mobile->>Server: POST /api/v1/auth/device-activate { userCode: "842-910" } (Authorization: Bearer <UserJWT>)
    Server-->>Mobile: 200 OK { pairedDevice: "Living Room Apple TV" }

    TV->>Server: POST /api/v1/auth/device-poll { deviceCode: "d8f9..." }
    Server-->>TV: 200 OK { status: "APPROVED", accessToken: "<TV_JWT>", localUrl: "http://192.168.1.50:8080", remoteUrl: "https://..." }
    TV->>TV: Stores Token & Launches 10-Foot Catalog Dashboard
```

### 4.2 Client-Aware 3-Tier Playback Decision Flow

```mermaid
sequenceDiagram
    autonumber
    participant Client as Client (Web / iOS / Android / Android TV / Apple TV)
    participant API as Spring Boot Controllers
    participant Decision as StreamingDecisionService
    participant Transcoder as TranscodingService (StructuredTaskScope)

    Client->>API: GET /api/v1/media/42 (Header: X-Locaflix-Client-Platform, X-Locaflix-Network-Mode)
    API->>Decision: evaluate(mediaItem=42, platform, isLan)

    alt Tier 1: Direct Play Supported (e.g., Android TV playing MKV/HEVC on LAN, or Web playing MP4/H264)
        Decision-->>API: DirectPlay(url="/api/v1/media/42/stream")
        API-->>Client: 200 OK { streamingMode: "DIRECT_PLAY", streamUrl: "/api/v1/media/42/stream" }
        Client->>API: GET /api/v1/media/42/stream?streamToken=... (Range: bytes=0-)
        API-->>Client: 206 Partial Content (4 MB ResourceRegion, 0% Server CPU)
    else Tier 2: Direct Stream Remux (e.g., Apple TV / iOS playing MKV with compatible HEVC/H264 video on LAN)
        Decision-->>API: DirectStreamRemux(url="/api/v1/media/42/hls/master.m3u8?mode=remux")
        API-->>Client: 200 OK { streamingMode: "DIRECT_STREAM_REMUX", streamUrl: ".../master.m3u8?mode=remux" }
        Client->>API: GET /api/v1/media/42/hls/master.m3u8?mode=remux&streamToken=...
        API->>Transcoder: Spawn ffmpeg -c:v copy -c:a aac -hls_segment_type fmp4 (<2% Server CPU)
        Transcoder-->>Client: Virtual VOD Playlist + lossless fMP4 video segments
    else Tier 3: Full HLS Transcode (WAN connection or incompatible video codec)
        Decision-->>API: HlsTranscode(url="/api/v1/media/42/hls/master.m3u8?mode=transcode")
        API-->>Client: 200 OK { streamingMode: "HLS_TRANSCODE", streamUrl: ".../master.m3u8?mode=transcode" }
        Client->>API: GET /api/v1/media/42/hls/master.m3u8?mode=transcode&streamToken=...
        API->>Transcoder: Spawn GPU-accelerated ffmpeg (-hwaccel + keyframe seek -ss)
        Transcoder-->>Client: Virtual VOD Playlist + adaptive .ts segments
    end
```

---

## 5. Non-Functional Requirements & Technical Constraints

### 5.1 Concurrency & Resource Management
- **Java 25 Virtual Threads**: Enabled globally (`spring.threads.virtual.enabled=true`) so concurrent streams across living-room TVs, mobile devices, and web browsers scale effortlessly without blocking OS threads.
- **Structured Subprocess Lifecycle**: Every `ffmpeg` and `ffprobe` invocation is wrapped in a `StructuredTaskScope` that drains `stdout`/`stderr` non-blockingly and destroys orphaned processes after 60 seconds of inactivity.
- **SQLite WAL Serialization**: Configured with `journal_mode=WAL`, `synchronous=NORMAL`, and `busy_timeout=5000` to handle simultaneous playback heartbeats from multiple family devices alongside background library ingestion.

### 5.2 TV & Mobile Specific Quality Attributes
- **10-Foot TV Ergonomics**: All screens in `apps/tv` enforce a `5%` overscan-safe margin, minimum `24sp` typography, explicit directional focus guides (`TVFocusGuideView`), high-contrast focus scaling (`1.08x`), and hardware remote media key bindings.
- **Cross-Device Continuity**: Stopping playback on any device flushes an immediate heartbeat so opening another device within 1 second reflects the updated resume point in "Continue Watching".

---

## 6. 4-Tier Testing & Verification Strategy

```mermaid
flowchart LR
    U["1. Unit Tests\n(JUnit 5 / Vitest across Server, SDK, Web, Mobile, TV)"] --> C["2. Component Tests\n(@WebMvcTest, @DataJpaTest / RTL & RNTL)"]
    C --> I["3. Integration Tests\n(@SpringBootTest + SQLite WAL + FFmpeg / MSW)"]
    I --> E["4. E2E Tests\n(Playwright for Web + Maestro/Detox for Mobile & TV)"]
```

1. **Unit Tests**: Validate per-platform `StreamingDecisionService` rules (`WEB`, `IOS`, `ANDROID`, `ANDROID_TV`, `APPLE_TV`), byte-range math, `FFprobe` JSON parsing, JWT/stream-token/device-code cryptography, and `core-sdk` LAN/WAN/mDNS resolution.
2. **Component-Level Tests**: Test Spring controllers (`@WebMvcTest`) and JPA repositories (`@DataJpaTest`); test Web components (`RTL`) and React Native Mobile/TV components (`@testing-library/react-native`), including TV D-pad focus transitions and PIN pairing UI.
3. **Integration Tests**: Full `@SpringBootTest` verification of SQLite WAL concurrency, `HTTP 206` byte-range streaming, `StructuredTaskScope` remux & transcode pipelines, TV device-code pairing workflow, and CORS/PNA security filters.
4. **End-to-End (E2E) Tests**: Playwright full-stack browser tests for Web + Server, paired with Maestro/Detox E2E flows for Mobile (iOS/Android) and TV (Android TV/Apple TV).
