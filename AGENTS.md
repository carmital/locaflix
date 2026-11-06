# Agent Rules & Technical Standards

## Monorepo & Tech Stack Guidelines
- **Repository Structure**: Unified Monorepo (`server/`, `packages/core-sdk/`, `apps/web/`, `apps/mobile/`, `apps/tv/`, `e2e/`).
- **Server (`server/`)**: Java 25 LTS, Spring Boot 4.1.x, Spring Data JPA, Spring Security, SQLite (WAL mode).
- Always use Java 25 Virtual Threads and modern Java constructs (records, sealed interfaces, pattern matching, `StructuredTaskScope`).
- **Shared Client SDK (`packages/core-sdk/`)**: Pure TypeScript package shared across Web, Mobile, and TV containing explicit API DTO interfaces, client capability profiles (`WEB`, `IOS`, `ANDROID`, `ANDROID_TV`, `APPLE_TV`), LAN/WAN connection manager, Zustand stores, and TanStack Query hooks.
- **Web Client (`apps/web/`)**: React 19, TypeScript, Vite, Tailwind CSS, Video.js / Hls.js, Lucide React.
- **Mobile Client (`apps/mobile/`)**: React Native (iOS & Android), TypeScript, NativeWind / StyleSheet, `react-native-video` (AVPlayer on iOS, Media3/ExoPlayer on Android), mDNS/Zeroconf discovery.
- **TV Client (`apps/tv/`)**: React Native TV (`react-native-tvos` for Android TV & Apple TV / tvOS), TypeScript, 10-foot spatial D-pad/Siri Remote focus navigation, QR/PIN device pairing, `react-native-video`.
- Always write type-safe TypeScript with explicit interfaces for API DTOs, client capability profiles, and player states.

## Architecture Guidelines
- **Direct Play First**: Never transcode if the target client platform (`WEB`, `IOS`, `ANDROID`, `ANDROID_TV`, `APPLE_TV`) can Direct Play. Direct Play must be served via Spring `ResourceRegion` with `HTTP 206` byte-range support (4 MB chunks).
- **Direct Stream (Remux) Before Transcode**: When a client (e.g., iOS/Apple TV `AVPlayer`) supports the video codec (H.264/HEVC) but not the source container (`.mkv`), use container remuxing (`-c:v copy`) to fMP4 HLS rather than re-encoding video.
- **Isolated Subprocess Transcoding**: All FFprobe and FFmpeg processes must be isolated to background subprocesses managed via Java 25 `StructuredTaskScope` with non-blocking stdout/stderr reading.
- **UX Paradigm**: All clients must remain responsive and dark-mode first, adhering to the Netflix/Plex browsing paradigm (touch/gesture optimized on Mobile, mouse/keyboard on Web, and 10-foot D-pad/Siri Remote focus navigation with QR/PIN login on Android TV & Apple TV).
- **Hybrid Networking**: All API and streaming communication must support both local LAN base URLs (via HTTP health probe and mDNS/Bonjour discovery) and remote HTTPS tunnel endpoints.

## Testing Guidelines
- Always cover the project comprehensively with automated tests across all four levels:
  - **Unit Tests**: Validate isolated business logic, domain models, per-platform streaming decisions, FFprobe parsers, and helper utilities (JUnit 5 + AssertJ/Mockito on the server; Vitest across `packages/core-sdk`, `apps/web`, `apps/mobile`, and `apps/tv`).
  - **Component-Level Tests**: Validate individual controllers, repositories, and Web/Mobile/TV UI & player components in isolation (Spring `@WebMvcTest` and `@DataJpaTest` slices on the server; React Testing Library / React Native Testing Library + Vitest on the clients).
  - **Integration Tests**: Validate multi-layer interactions including SQLite (WAL) persistence, `ResourceRegion` HTTP 206 byte-range streaming, FFmpeg/FFprobe subprocess pipelines, TV device-code pairing, and Spring Security JWT/CORS/PNA filters (`@SpringBootTest` + MockMvc; MSW integration tests for `core-sdk` and client stores).
  - **End-to-End (E2E) Tests**: Validate complete user journeys across the stack (Playwright full-stack tests for Web + Server covering login, catalog browsing, LAN/WAN switching, Direct Play & HLS playback, and progress tracking; Maestro/Detox E2E test suites for Mobile and TV navigation/pairing).
- Every feature, endpoint, or bug fix must include corresponding unit, component, integration, and E2E test coverage.

## Task Tracking & Git Commit Guidelines
- All implementation work must follow the tracked task backlog in `TASKS.md` (`LFX-001` through `LFX-040`).
- Complete one task at a time in dependency order (unless parallelized on isolated branches), verify all required tests pass, update the task status in `TASKS.md` (`[x]`), and create a dedicated Git commit marked with the task ID.
- **Mandatory Commit Message Format**: `[LFX-XXX] <type>(<scope>): <description>` (e.g., `[LFX-008] feat(server): implement 3-tier StreamingDecisionService with unit tests`).
