# Unity VGF Client Technical Design Document

## Summary
- Run the Unity runtime on AWS GameLift Streams and deliver the rendered experience to Smart TVs and streaming devices.
- Port the existing web-based VGF client to a Unity-first implementation that runs within the streamed session whilst preserving device-specific UX.
- Maintain feature parity with the current PartyTime client: real-time state synchronisation, reducer/thunk dispatch, lifecycle events, and session management.
- Support voice input from Smart TV remotes and mobile companion devices, streaming audio to the VGF backend for command interpretation.
- Deliver a reusable Unity package that game teams can embed in either 2D or 3D projects.

## Scope
- Implement a managed C# SDK that mirrors the behaviour of `@volley/vgf`’s PartyTime client and Socket.IO transport.
- Provide Unity components, services, and sample scenes to accelerate adoption.
- Define contracts with the Smart TV streaming client (GameLift Streams) for session lifecycle, telemetry, and input delivery while keeping Unity agnostic of GameLift specifics.
- Create platform adapters for voice capture (Smart TV remote, iOS/Android companion app).
- Establish automated build, test, and release workflows for the Unity client, including GameLift-friendly build artefacts and streamed-session telemetry hooks.

## Out of Scope
- Rewriting backend services beyond required configuration for Unity clients and GameLift match/session orchestration.
- Building production voice-to-intent models (the backend already handles transcription/intent).
- Creating final game-specific UI; the SDK delivers integration hooks and sample prefabs only.

## Assumptions
- Backend functionality, including session lifecycle and state broadcasting, stays as implemented in `/Users/pratik/dev/vgf`.
- Socket.IO remains the transport protocol between client and server (`SocketIOTransport` in the backend).
- AWS GameLift Streams hosts the Unity build; Smart TV/streaming apps handle session negotiation using the GameLift Streams SDK (`/Users/pratik/dev/ccm/cocomelon-hub` reference).
- Unity projects target .NET 6-compatible scripting backend (IL2CPP/Mono) whilst running in the GameLift Streams environment.
- GameLift orchestration supplies signed session metadata to the Smart TV app, which in turn provides `sessionId`, `userId`, and `clientType` to the Unity runtime (e.g. via query params or environment variables) before the VGF connection is initialised.
- Mobile companion app is available and already capable of streaming audio to VGF when provided session credentials.
- Smart TV devices use the GameLift Streams client SDK for video playback and input relay; Unity consumes injected input events but does not talk to the GameLift API directly.

## Component Placement
```
+------------------------------+  WebRTC / REST  +----------------------+  Socket.IO / REST  +------------------------------+
|  Smart TV Streaming Client   |<--------------->|   Unity Runtime      |<------------------->|         VGF Backend          |
| (GameLift Streams SDK, e.g.  |                |  (GameLift host VM)  |                    |  (Node.js / Redis / REST)    |
|  cocomelon-hub)              |                |                      |                    |                              |
|                              |                |  [UnityVGFContext]   |                    |  [SocketIOTransport]         |
|  [GameLiftStreams wrapper]   |                |  [UnityPartyTimeClient]|                   |  [FailoverVGFServiceFactory] |
|  [Start/Poll/End session API]|                |  [UnityVGFTransport]  |                    |  [PartyTimeServer]           |
|  [Telemetry overlay]         |                |  [StateSync Layer]    |                    |  [SessionScheduler/Redis]    |
|  [QR Pairing UI]             |                |  [VoiceInputService]* |                    |  [Reducers/Thunks]           |
|  [Companion bridge]*         |                |  [Sample Scenes]      |                    |  [State Update Broadcast]    |
|                              |                |                      |                    |  [Voice Processing Service]* |
|  (Maps device input & voice) |                |  (Receives injected   |                    |  (Session orchestration APIs) |
|                              |                |   input channels)     |                    |                              |
+------------------------------+                +----------------------+                    +------------------------------+

* Voice pipeline may span Unity runtime and backend services depending on implementation.
```

## Existing System Overview
The JavaScript PartyTime client drives today’s browser games via `SocketIOClientTransport` and `PartyTimeClient`.

```56:285:/Users/pratik/dev/vgf/packages/vgf/src/client/PartyTimeClient/PartyTimeClient.ts
// ... message dispatch logic mirrored by Unity client ...
```

Key behaviours to replicate:
- Query-string based session/user bootstrap before connecting (`sessionId`, `userId`, `clientType`).
- Dispatch pathways for reducers and thunks with acknowledgement time-outs.
- Event-driven state updates via `StateUpdateMessage` payloads emitted by the backend.

`SocketIOClientTransport` wraps `socket.io-client` and exposes middleware hooks.

```32:143:/Users/pratik/dev/vgf/packages/vgf/src/client/transport/SocketIOClientTransport/SocketIOClientTransport.ts
// ... connection handling and middleware pipeline ...
```

The server expects compliance with `MessageType` contracts and `StateUpdateMessageSchema`.

```1:12:/Users/pratik/dev/vgf/packages/vgf/src/server/schemas/StateUpdateMessageSchema.ts
// ... schema for state update messages ...
```

`/Users/pratik/dev/ccm/cocomelon-hub` demonstrates the GameLift Streams Smart TV application that negotiates stream sessions and exposes telemetry; Unity must cooperate with that client but remain unaware of the underlying GameLift APIs.

## Proposed Architecture
### High-level Component Layout
- **UnityVGFTransport**: C# Socket.IO wrapper that mirrors the TypeScript transport API, exposing `Connect`, `Disconnect`, `Emit`, and event hooks.
- **UnityPartyTimeClient**: Event-based client that maintains session state, handles reducer/thunk dispatch, and raises Unity events for state changes.
- **StateSynchronisationLayer**: Serialises/deserialises `StateUpdateMessage` payloads into strongly-typed Unity data models.
- **UnityVGFContext**: ScriptableObject-based store and dependency injection container that Unity scenes can bind to.
- **InputAdapter**: Lightweight abstraction that maps GameLift-injected input or companion signals onto Unity systems and VGF dispatches.
- **VoiceInputService**: Abstract interface with platform-specific implementations for Smart TV remotes and mobile companion bridging.
- **CompanionPairingService**: Displays QR codes, registers companion devices, and relays voice session tokens.
- **Sample Integrations**: 2D and 3D demo scenes showing state-driven UI, voice command reactions, reconnection flows, with development overlays akin to `StreamStatsOverlay`.

```
+-------------------+       +------------------------+
| Unity Game Scene  |<----->| UnityVGFContext        |
| (2D/3D)           |       |  - State store         |
| - UI Layer        |       |  - Dispatcher façade   |
+-------------------+       +-----------+------------+
                                        |
                 +----------------------+--------------------+
                 |                                           |
       +---------v----------+                 +--------------v-------------+
       | UnityPartyTimeClient|<--->transport-->| UnityVGFTransport          |
       | - Reducer dispatch  |                 | - Socket.IO connection     |
       | - State cache       |                 | - Middleware hooks         |
       +---------+-----------+                 +--------------+-------------+
                 |                                               |
       +---------v-----------+                     +-------------v--------------+
       | StateSync Layer     |                     | Voice & Companion Services |
       | - JSON serialisation|                     | - VoiceInputService        |
       | - Phase events      |                     | - CompanionPairingService  |
       +---------+-----------+                     +-------------+--------------+
                 |
       +---------v-----------+
       | Input Adapter       |
       | - Maps injected     |
       |   GameLift inputs   |
       |   & companions      |
       +---------------------+
```

## Detailed Design
### Smart TV Streaming Client (Outside Unity)
- The Smart TV or streaming-device application (e.g. `/Users/pratik/dev/ccm/cocomelon-hub`) owns GameLift Streams responsibilities:
  - Instantiate `GameLiftStreams` with HTML video/audio elements.
  - Call `generateSignalRequest()`, send to `createStreamSession`, poll `pollForSignalResponse`, and invoke `processSignalResponse()`.
  - Display telemetry overlays (`StreamStatsOverlay`) and developer controls.
  - Surface query parameters or signed payloads (session/user IDs) to the Unity runtime via URL, environment variables, or IPC to initialise the VGF client.
  - Relay device inputs and microphone toggles through GameLift’s input module so Unity receives standard input events.
  - Handle session teardown (`deleteStreamSession`) and error reporting without requiring Unity changes.
- Unity provides documentation/specs describing expected parameters and telemetry callbacks so the Smart TV client can integrate smoothly.

### UnityVGFTransport
- Built over `socket.io-client-csharp` (Unity-compatible) with IL2CPP support.
- Exposes:
  - `Task ConnectAsync(SessionMemberState? state = null)` – aligns with `connect` semantics.
  - `void Disconnect()`.
  - `void Emit<TEvent>(string eventName, params object[] args)` with acknowledgement callbacks.
  - `void On<TEvent>(string eventName, Action<TEvent> handler)`.
- Applies middleware-based pipelines using composable delegates to emulate incoming/outgoing middleware.
- Maintains reconnection jitter and back-off matching current defaults (5 attempts, 3–6s delays).
- Emits diagnostics events for logging and metrics.

### UnityPartyTimeClient
- Implements the PartyTime protocol in C# using `UnityEvent`/C# events for consumers.
- Holds the latest `Session` object; on `StateUpdateMessage`, updates cache then raises `OnSessionUpdated`.
- Dispatch Methods:
  - `Task DispatchReducerAsync(string reducerName, object[] args, TimeSpan timeout)`.
  - `Task DispatchThunkAsync(string thunkName, object[] args, TimeSpan timeout)`.
  - Both wrap `Emit` with acknowledgement callbacks mirroring failure paths in TypeScript (`ActionFailedError`, `DispatchTimeoutError`).
- Errors propagate via `OnError` event with typed error objects.
- Supports lifecycle events (connect/disconnect/reconnect) for Unity UI binding.

### State Synchronisation Layer
- Defines DTOs mirroring backend schemas (generated from shared JSON Schema or manual mapping).
- Uses `System.Text.Json` with source generators for IL2CPP friendliness.
- Exposes `IGameStateAdapter<TState>` for game-specific state mapping.
- Supports phase-specific reducers/thunks by exposing metadata from `StateUpdateMessage`.

### UnityVGFContext
- ScriptableObject storing configuration (backend endpoint, logger levels, default time-outs, session identifiers passed in by the Smart TV client).
- Provides DI container for Unity components via `[CreateAssetMenu]` pattern.
- Components fetch services through `VGFContextBehaviour` base class, ensuring testability.

### Input Adapter
- Reads device input delivered via GameLift’s input injection (e.g. standard Unity `Input` events or native plugin callbacks).
- Maps recognised gestures/buttons to reducer or thunk dispatches through `UnityPartyTimeClient`.
- Exposes hooks for companion app signals (e.g. REST/WebSocket callbacks that the Smart TV client forwards).

### VoiceInputService
- Interface: `StartCapture`, `StopCapture`, `OnChunkAvailable`, `OnError`.
- **Smart TV Implementation**: Unity listens for audio streams forwarded by the Smart TV client (GameLift audio channel or dedicated WebRTC); service parses chunks and forwards to backend endpoints.
- **Mobile Companion Bridge**: Unity scene renders QR containing sessionId/userId; when mobile app pairs, it streams audio to backend using existing APIs. Unity client only needs to expose handshake endpoint, not audio streaming directly.
- Streams audio via WebRTC or secure WebSocket to backend voice service; reuse existing backend endpoints.
- Provides transcription events back to Unity for UX feedback.

### CompanionPairingService
- Generates session-specific QR codes pointing to the pairing endpoint (HTTPS) with query parameters.
- Subscribes to backend pairing events (through VGF server or separate REST polling) to confirm companion availability.
- Maintains mapping of `userId -> deviceType` to enforce `RoomFullError` logic consistent with backend.

### Sample Scenes & Prefabs
- `Scenes/Display2D.unity`: Canvas-based UI reacting to state changes via binding scripts.
- `Scenes/Display3D.unity`: 3D environment using ScriptableRenderPipeline example.
- Prefabs: `VGFConnectionStatus`, `VGFRosterList`, `VGFAudioIndicator`, `StreamMetricsOverlay` hooks.
- Provide `MonoBehaviour` scripts demonstrating reducer/thunk dispatch for input events and collaboration with Smart TV telemetry callbacks.

### Configuration & Build
- Package delivered as UPM package (`com.volley.vgf.unity`) with asmdef for runtime/editor separation.
- CI builds UPM tarball, runs EditMode/PlayMode tests and mock integration suite.
- Produce GameLift-compatible headless build using Unity Build Pipeline and package into container image (Unity remains oblivious to GameLift APIs).
- Support `Development` and `Production` configuration assets with environment URLs, session identifier injection strategy, and telemetry flags (e.g. first-frame timeouts, manual start toggles).

## Data Flow
### Connection bootstrap
1. Smart TV application launches GameLift Streams client, negotiates the streaming session (create/poll/process) and, upon success, injects session metadata (`sessionId`, `userId`, `clientType`) into the Unity runtime (e.g. query params on the hosted page).
2. Unity runtime starts within the GameLift host, loads `UnityVGFContext`, and reads injected metadata.
3. `UnityVGFTransport` prepares Socket.IO connection with the provided parameters.
4. Transport connects; server middlewares (`roomFullMiddleware`, Datadog) validate request.
5. On success, `UnityPartyTimeClient` requests initial state.
6. `StateSync` layer updates caches; Unity UI refreshes and frames are streamed back to the Smart TV.

### Reducer Dispatch Sequence
```
Smart TV Input (delivered via GameLift injection)
  -> Input Adapter maps to gameplay action
    -> UnityPartyTimeClient.DispatchReducerAsync
      -> UnityVGFTransport.Emit("message", reducer payload)
        -> Server processes reducer
          -> State update broadcast (StateUpdateMessage)
            -> UnityVGFTransport receives
              -> UnityPartyTimeClient.UpdateSession
                -> StateSync raises OnStateChanged
                  -> Game scripts read latest state
                  -> Rendered frame streamed via GameLift
```

### Voice Command Flow
1. Player holds microphone button on remote (or companion app); Smart TV client forwards audio stream.
2. `VoiceInputService` in Unity relays audio to backend voice endpoint (or the Smart TV client sends audio directly while Unity handles command feedback).
3. Backend transcribes, converts to intent, dispatches relevant reducer/thunk.
4. Resulting state update flows back through standard pipeline.
5. Unity UI optionally displays transcription feedback, rendered in the stream.

## Error Handling & Resilience
- Transport reconnection with exponential back-off; after max attempts, surfaces UI prompt.
- Smart TV client monitors GameLift Streams session health; Unity handles loss-of-input by pausing gameplay or displaying status overlays.
- Distinguish backend errors (`RoomFullError`, `UnauthorizedError`) with user-friendly messaging in-stream.
- Guard IL2CPP compatibility through AOT-friendly patterns (avoid reflection-heavy code).
- Offline caching: maintain last-known state, allow limited local interactions (queued dispatches) with eventual reconciliation.
- Ensure voice capture stops on disconnect to avoid dangling native resources.

## Observability
- Integrate ILogger-compatible interface with sinks to Unity Console and optional Datadog via HTTP.
- Expose hooks so the Smart TV client can log timing marks (API, polling, WebRTC, first frame) and surface them in Unity debug overlays.
- Emit structured logs for connection lifecycle, reducer dispatch timings, voice session events, and collaboration signals from the Smart TV client.

## Security & Privacy
- Enforce TLS for all transport connections between Unity and backend.
- Do not persist raw audio; stream directly to backend and discard buffers post-send.
- Obfuscate QR codes with short-lived pairing tokens rather than raw session IDs.
- Adhere to platform microphone permission prompts and display usage indicators.
- Protect session credentials; Smart TV client rotates tokens and provides Unity with only the values needed for VGF connection.

## Testing Strategy
- **Unit Tests (EditMode)**: Mock transport to verify reducer/thunk dispatch, state parsing, error propagation.
- **Integration Tests (PlayMode)**: Spin up local Node-based VGF server (Docker) and validate end-to-end state synchronisation.
- **Smart TV Contract Tests**: Use harness mirroring `cocomelon-hub` to ensure Unity correctly consumes injected parameters and input events.
- **Voice Pipeline Tests**: Use prerecorded audio chunks with a stub backend to ensure streaming API compliance.
- **Contract Tests**: Validate serialisation against `StateUpdateMessageSchema` via generated fixtures.
- **Load & Resilience**: Simulate network drop-outs, reconnection storms, GameLift session restarts, and rapid voice toggling; verify telemetry marks remain coherent.
- **Cross-Platform Smoke**: Automated builds for FireTV (Android), tvOS, and Tizen via cloud build matrix with GameLift clients.

## Migration Plan
1. Deliver Unity SDK alpha with reducer/thunk support, Smart TV parameter integration, and sample scene.
2. Integrate with existing backend QA environment; run parity tests versus web client through local streaming clients.
3. Expand to include voice capture integration; validate on hardware labs and GameLift staging environments.
4. Harden reconnection/error flows; add observability tooling (Smart TV telemetry, Datadog dashboards, in-stream overlays).
5. Release v1.0 UPM package and Smart TV integration guide; deprecate web client in chosen pilot title once Unity client proves stable.

## Open Questions
- Confirm preferred C# Socket.IO library for long-term support and IL2CPP compatibility.
- Clarify backend expectations for audio streaming (codec, endpoint, authentication) within Smart TV/GameLift constraints.
- Determine whether Unity client must handle session auto-creation (`autoCreateSession`) or leave to backend orchestration (current assumption: orchestration supplies session metadata; Unity should only join).
- Validate target Smart TV platforms, available GameLift client SDKs, and microphone channel capabilities.
- Agree on GameLift fleet scaling strategy and cost monitoring requirements.
