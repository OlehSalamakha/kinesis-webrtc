# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project overview

Amazon Kinesis Video Streams C WebRTC SDK — a pure C (C99) WebRTC client library implementing signaling, ICE/STUN/TURN, DTLS-SRTP, RTP/RTCP, and SCTP data channels, purpose-built for embedded/IoT use with AWS Kinesis Video Streams as the signaling and media-storage backend. Depends on two AWS submodules pulled under `open-source/`: `amazon-kinesis-video-streams-pic` (platform-independent code: threading, logging, allocators) and `amazon-kinesis-video-streams-producer-c`.

## Build

```shell
git submodule update --init --recursive   # if submodules aren't populated
mkdir -p build && cd build
cmake ..
make -j
```

By default CMake downloads and builds all third-party dependencies (OpenSSL 1.1.1t, libwebsockets, libsrtp, libusrsctp) from source (`-DBUILD_DEPENDENCIES=ON`). To link against system-installed dependencies instead: `cmake .. -DBUILD_DEPENDENCIES=OFF -DUSE_OPENSSL=ON` (or `-DUSE_MBEDTLS=ON` for mbedTLS). Requires CMake 3.x (CMake 4.x not yet supported) and `pkg-config`.

Notable CMake flags (see README.md "CMake Arguments" for the full list):
- `-DBUILD_TEST=TRUE` — build the GTest suite (`./tst/webrtc_client_test`)
- `-DBUILD_SAMPLE` — build sample executables (ON by default)
- `-DUSE_MBEDTLS=ON` / `-DUSE_OPENSSL=ON` — crypto backend selection (mutually exclusive; OpenSSL is default). Every crypto-touching source file has `_openssl.c` / `_mbedtls.c` variants (e.g. `Dtls_openssl.c`/`Dtls_mbedtls.c`, `Tls_openssl.c`/`Tls_mbedtls.c`) selected at build time — changes to crypto logic usually need both implementations updated in lockstep.
- `-DUSE_MBEDTLS4=ON` — mbedTLS 4.x line (implies `-DUSE_LIBSRTP3=ON`); requires Python 3 with `jinja2`/`jsonschema` at configure time
- `-DENABLE_DATA_CHANNEL` — SCTP data channel support (ON by default)
- `-DENABLE_KVS_THREADPOOL` — threadpool for signaling (OFF by default)
- `-DBUILD_STATIC_LIBS` — static build of SDK + all deps
- `-DCOMPILER_WARNINGS`, `-DADDRESS_SANITIZER`, `-DTHREAD_SANITIZER`, `-DUNDEFINED_BEHAVIOR_SANITIZER`, `-DMEMORY_SANITIZER`, `-DCODE_COVERAGE` — quality/CI tooling flags
- `-DMAX_SDP_SESSION_MEDIA_COUNT`, `-DMAX_SDP_ATTRIBUTES_COUNT`, `-DKVS_SIGNALING_MESSAGE_LEN` — compile-time buffer sizing (see `docs/SDP_LIMITS.md`)

On macOS M1, if OpenSSL/websockets fail to build: `cmake .. -DBUILD_OPENSSL_PLATFORM=darwin64-arm64-cc`.

## Test

Build with `-DBUILD_TEST=TRUE`, then run the GTest binary from the build directory:

```shell
./tst/webrtc_client_test
./tst/webrtc_client_test --gtest_filter=IceAgentFunctionalityTest.*   # single suite
./tst/webrtc_client_test --gtest_filter=*SpecificTestName*            # single test
```

Test files live in `tst/` (one `*Test.cpp` per subsystem, e.g. `IceAgentFunctionalityTest.cpp`, `PeerConnectionApiTest.cpp`, `SdpFunctionalityTest.cpp`, `DtlsFunctionalityTest.cpp`, `SignalingApiTest.cpp`). Many functional tests connect to a real AWS account/signaling channel and require AWS credentials in the environment (see Run section below); pure unit-style tests (Sdp, Stun, Rtcp, Rtp parsing, etc.) do not. `tst/suppressions/` holds LSAN/TSAN suppression files used in sanitizer CI runs.

## Lint / format

The project enforces `clang-format` (Chromium-based style, see `.clang-format`) on all `.c`/`.h` files under `src`, `samples`, `tst` (excluding `*/external/*`):

```shell
scripts/check-clang.sh     # CI check, exits non-zero on violations
scripts/clang-format.sh    # auto-fixes formatting in place
```

## Running samples

After building with `-DBUILD_SAMPLE=ON` (default), from `build/`:

```shell
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-west-2   # optional, defaults to us-west-2
export AWS_KVS_LOG_LEVEL=2            # optional, DEBUG; default is WARN (4)

./samples/kvsWebrtcClientMaster <channelName> [mediaType] [audio-codec] [video-codec]
./samples/kvsWebrtcClientViewer <channelName> ...
```

Master sends bundled sample H264/Opus/H265 frames from `samples/h264SampleFrames`, `samples/opusSampleFrames`, `samples/h265SampleFrames`. See README.md "Running the Samples" for the full set of peer-to-peer, WebRTC-storage, and IoT-credential sample variants and their environment variables.

## Architecture

### Layering

```
samples/                         -- master/viewer/p2p demo apps, GStreamer capture glue
  src/include/.../Include.h      -- the entire public C API surface (structs, enums, PUBLIC_API fns)
    src/source/PeerConnection/   -- PeerConnection.c: top-level session orchestration, SDP O/A glue
        ├── SessionDescription.c -- SDP parsing/serialization/munging
        ├── DataChannel.c        -- SCTP-backed RTCDataChannel
        ├── Rtp.c / Rtcp.c       -- packetization, sender/receiver reports, NACK/RTX (see docs/NACK_RTX.md)
        ├── Retransmitter.c      -- RTX bookkeeping
        └── JitterBuffer.c
    src/source/Ice/               -- ICE agent: candidate gathering, STUN/TURN connectivity checks
        ├── IceAgent.c + IceAgentStateMachine.c   -- ICE state machine (gathering -> checking -> connected)
        ├── TurnConnection.c + TurnConnectionStateMachine.c
        ├── SocketConnection.c, ConnectionListener.c, Network.c -- non-blocking socket I/O, multiplexed receive loop
        └── NatBehaviorDiscovery.c
    src/source/Signaling/          -- AWS KVS signaling channel client (control plane)
        ├── Client.c              -- public signaling client entry points
        ├── LwsApiCalls.c         -- libwebsockets-based HTTPS/WSS calls to the KVS signaling service
        ├── StateMachine.c        -- signaling client connection state machine
        ├── ChannelInfo.c         -- channel ARN/endpoint/caching-policy resolution
        └── FileCache.c           -- on-disk cache for DescribeSignalingChannel/GetSignalingChannelEndpoint
    src/source/Crypto/             -- Dtls.c/Tls.c dispatch to *_openssl.c or *_mbedtls.c backend at build time
    src/source/Srtp/                -- libsrtp wrapper for media encryption
    src/source/Sctp/                -- libusrsctp wrapper for data channels
    src/source/Stun/                -- STUN message encode/decode (RFC 5389/8489)
    src/source/Rtcp/RollingBuffer.c, RtpRollingBuffer.c -- circular buffers for RTX/jitter handling
    src/source/Rtp/Codecs/          -- per-codec RTP payloaders: H264, H265, VP8, Opus, G711
    src/source/Threadpool/          -- optional shared threadpool (ENABLE_KVS_THREADPOOL)
    src/source/Metrics/             -- ICE/peer connection stats aggregation (W3C webrtc-stats shaped)
```

`src/source/Include_i.h` is the internal umbrella header (pulled in by nearly every `.c` file) — it wires in the public `Include.h`, the PIC headers, and OpenSSL-vs-mbedTLS conditional includes. New internal-only helpers/macros typically belong here or in a subsystem's own `_i.h`/private header, not in `Include.h`.

### Key patterns

- **State machines everywhere**: ICE connectivity (`IceAgentStateMachine.c`), TURN allocation (`TurnConnectionStateMachine.c`), and the signaling client (`StateMachine.c`) are all driven by explicit state-machine structs from PIC's state machine utility, not ad hoc flags. When modifying connection-establishment behavior, find the relevant state table first.
- **Dual crypto backends**: almost every file in `src/source/Crypto/` and callers of it must work correctly whether built with OpenSSL or mbedTLS — check both `_openssl.c` and `_mbedtls.c` implementations before considering a crypto-related change complete.
- **PUBLIC_API surface**: all externally-callable functions are declared in `src/include/.../Include.h` and marked `PUBLIC_API`. This is the contract samples and downstream consumers depend on — changing signatures here is a breaking change to the SDK's ABI/API.
- **Handles over pointers at the API boundary**: peer connections, signaling clients, etc. are referenced by opaque `UINT64` handles (e.g. `SIGNALING_CLIENT_HANDLE`) or forward-declared struct pointers (`PRtcPeerConnection`) rather than exposing internal structs.
- **STATUS return convention**: nearly every internal and public function returns a `STATUS` code (`STATUS_SUCCESS` / error codes defined in PIC); errors propagate via `CHK`-style macros rather than exceptions (this is C).
- **Caching**: the signaling client supports pluggable caching of `DescribeSignalingChannel`/`GetSignalingChannelEndpoint`/`DescribeMediaStorageConfiguration` responses via `ChannelInfo.cachingPolicy` (`FileCache.c` for on-disk persistence) — relevant when touching signaling connection setup, see README "Signaling API call caching".

### Submodule dependency (PIC)

Fundamental types (`STATUS`, `THREAD_CREATE`, allocators, logging macros, state machine utilities, stack queue/hashtable data structures) come from the `amazon-kinesis-video-streams-pic` submodule under `open-source/`, not from this repo. When a symbol/macro can't be found in `src/`, check PIC's headers before assuming it's missing.

## CI

GitHub Actions workflows (`.github/workflows/`) run cross-compiled builds, clang-format checks, sanitizer builds (ASAN/TSAN/UBSAN/MSAN), and functional tests against live AWS KVS signaling channels (skipped on forks without AWS credentials configured via OIDC). Some workflows specifically toggle `KVS_DUALSTACK_ENDPOINTS` and media-storage settings to validate the signaling API response cache behaves correctly across config changes.

## Contribution branch flow

PRs should be based on and target `origin/develop`, not `main`. `develop` is periodically merged into `main` as part of release cuts (see README "Development" and `CONTRIBUTING.md`).
