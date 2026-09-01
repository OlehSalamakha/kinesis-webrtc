# Building with Docker

## Why

This SDK builds several dependencies from source with autotools/CMake (OpenSSL, libsrtp, libusrsctp, libwebsockets, plus the PIC/producer-c submodule projects). Standing up a matching native cross-toolchain for a different target architecture — e.g. building on macOS/Intel for an arm64 Linux single-board computer — is possible (see [Cross-Compilation](../README.md#cross-compilation)) but fiddly, since some of those dependency build scripts assume a Linux-like host.

The more reliable option is to build inside a Docker container for the *target* platform. Docker Desktop transparently picks the right execution path:
- **Apple Silicon (arm64) Mac building for `linux/arm64`** — the container runs natively, no emulation, full speed.
- **Intel Mac, or targeting an architecture that doesn't match your host** — Docker Desktop uses QEMU (via binfmt_misc) to emulate the container's CPU. This works but is noticeably slower, especially for the from-source dependency builds (OpenSSL in particular).

This doc walks through building the SDK and samples for an aarch64 Linux target (the example throughout is an Orange Pi Zero 3 — Allwinner H618, quad-core Cortex-A53, typically running Armbian/Debian Bookworm or Ubuntu) using this approach. The same pattern works for other targets by changing the `--platform` flag and base image.

## Prerequisites

- Docker Desktop installed and running (`docker info` should succeed, not error with "Cannot connect to the Docker daemon").
- No cross-toolchain, emulator, or extra Homebrew packages needed on the host — everything happens inside the container.

## 1. Build the builder image

`docker/Dockerfile.aarch64` defines a Debian Bookworm (arm64) image with the toolchain and system packages that satisfy the SDK's [dependency requirements](../README.md#dependency-requirements) directly from apt (`libssl-dev`, `libsrtp2-dev`, `libusrsctp-dev`) — Debian's packaged `libwebsockets-dev` (4.1.6) is below the SDK's minimum (4.2.0), so that one is still built from source via `-DBUILD_WEBSOCKETS=ON`.

```shell
docker build --platform linux/arm64 -t kvs-webrtc-opi-builder -f docker/Dockerfile.aarch64 .
```

This only needs to be re-run when `docker/Dockerfile.aarch64` changes (new packages, base image bump, etc.) — Docker layer caching makes repeat builds fast.

To target a different Debian/Ubuntu-based board, adjust `FROM` in the Dockerfile and the `--platform` value (e.g. `linux/arm/v7` for a 32-bit ARMv7 board) to match.

## 2. Configure

Run `cmake` inside the container with the repository mounted, so build output lands in `build/` on the host (not inside the ephemeral container):

```shell
docker run --rm --platform linux/arm64 \
  -v "$(pwd)":/workspace -w /workspace \
  kvs-webrtc-opi-builder bash -c "
    mkdir -p build && cd build && \
    cmake .. \
      -DBUILD_DEPENDENCIES=OFF \
      -DBUILD_WEBSOCKETS=ON \
      -DBUILD_SAMPLE=ON \
      -DBUILD_STATIC_LIBS=ON \
      -DPARALLEL_BUILD=ON
  "
```

Flags used here, and why:

| Flag | Why |
|---|---|
| `-DBUILD_DEPENDENCIES=OFF` | Use the apt-installed OpenSSL/libsrtp2/libusrsctp instead of rebuilding them from source — much faster, especially under QEMU emulation. |
| `-DBUILD_WEBSOCKETS=ON` | Overrides the above for libwebsockets specifically, since Debian's package is too old (4.1.6 < required 4.2.0). |
| `-DBUILD_SAMPLE=ON` | Build the sample executables (default is already ON; listed for clarity). |
| `-DBUILD_STATIC_LIBS=ON` | Statically link the KVS/PIC/producer-c/libwebsockets/usrsctp libraries into the sample binaries, so only `libsrtp2` and glibc need to exist on the target device — avoids shipping/matching a pile of `.so` versions. |
| `-DPARALLEL_BUILD=ON` | Build dependency ExternalProjects with multiple cores. |

Add `-DIOT_CORE_ENABLE_CREDENTIALS=ON` if the samples should authenticate via an IoT Core device certificate (X.509 + role alias) instead of static `AWS_ACCESS_KEY_ID`/`AWS_SECRET_ACCESS_KEY` — see [Setup IoT](../README.md#setup-iot).

`cmake` auto-detects the system OpenSSL is >= 3.0 and enables `USE_OPENSSL3` accordingly; no need to pass it explicitly on Debian Bookworm.

## 3. Build

```shell
docker run --rm --platform linux/arm64 \
  -v "$(pwd)":/workspace -w /workspace \
  kvs-webrtc-opi-builder bash -c "cd build && make -j\$(nproc)"
```

Since `build/` is a bind mount, this is a normal incremental `make` after the first run — only rerun `cmake` (step 2) if you change CMake flags, or delete `build/` for a totally clean build.

## 4. Verify the output

```shell
file build/samples/kvsWebrtcClientMaster
# ELF 64-bit LSB pie executable, ARM aarch64, ... dynamically linked ...

docker run --rm --platform linux/arm64 -v "$(pwd)":/workspace -w /workspace \
  kvs-webrtc-opi-builder objdump -p build/samples/kvsWebrtcClientMaster | grep NEEDED
#   NEEDED   libsrtp2.so.1
#   NEEDED   libc.so.6
```

With `-DBUILD_STATIC_LIBS=ON`, the only non-libc runtime dependency should be `libsrtp2.so.1` (it comes from the apt package, not built from source, so it stays dynamic). That package is present in Armbian/Debian/Ubuntu's default repos for the target board.

You can also smoke-test the binary directly in the container (native execution on an arm64 host, QEMU-emulated on an Intel host) to confirm the dynamic linker resolves everything:

```shell
docker run --rm --platform linux/arm64 -v "$(pwd)":/workspace -w /workspace \
  kvs-webrtc-opi-builder bash -c "cd build/samples && ./kvsWebrtcClientMaster"
# should fail with a clean "AWS_ACCESS_KEY_ID must be set" rather than a linker error
```

## 5. Deploy to the device

`cmake` bakes the default CA cert path in as an **absolute path from the build container** (`KVS_CA_CERT_PATH=/workspace/certs/cert.pem`, since the repo was mounted at `/workspace`) — that path will not exist on the target device, so it must be overridden at runtime via `AWS_KVS_CACERT_PATH` rather than relied on as a default. The per-codec sample frame folders (`h264SampleFrames/`, `h265SampleFrames/`, `opusSampleFrames/`) are read via a relative `./` path, so they need to sit as siblings of the binary in whatever directory you run it from — copying `build/samples/` wholesale (as the build already lays them out) is the simplest way to keep that intact.

```shell
scp -r build/samples/ orangepi@<device-ip>:~/kvs-webrtc/
scp certs/cert.pem orangepi@<device-ip>:~/kvs-webrtc/
```

On the device, install the one remaining shared dependency and run from inside that copied directory:

```shell
sudo apt-get install -y libsrtp2-1
cd ~/kvs-webrtc
export AWS_KVS_CACERT_PATH=~/kvs-webrtc/cert.pem
export AWS_ACCESS_KEY_ID=...
export AWS_SECRET_ACCESS_KEY=...
export AWS_DEFAULT_REGION=us-west-2
./kvsWebrtcClientMaster <channelName>
```

If you built with `-DIOT_CORE_ENABLE_CREDENTIALS=ON`, swap the AWS key env vars for `AWS_IOT_CORE_CERT`, `AWS_IOT_CORE_PRIVATE_KEY`, `AWS_IOT_CORE_THING_NAME`, `AWS_IOT_CORE_ROLE_ALIAS`, and `AWS_IOT_CORE_CREDENTIAL_ENDPOINT` (your own IoT device cert/key files, provisioned separately — not part of this build).

See the README's [Running the Samples](../README.md#running-the-samples) and [Run](../README.md#run) sections for the full set of environment variables and sample variants.

## Notes

- **Building selective components** — the same container works for `-DBUILD_TEST=TRUE` (GTest suite) or `-DBUILD_BENCHMARK=TRUE`, though the test binary pulls in the AWS SDK for C++ and GTest, which are not installed by `docker/Dockerfile.aarch64`; extend the Dockerfile if you need to cross-build tests.
- **GStreamer samples** are skipped by default in this setup (`gstreamer-1.0` not found) since headless SBCs like the Orange Pi Zero 3 usually don't need the capture-from-webcam sample path. Add `libgstreamer1.0-dev libgstreamer-plugins-base1.0-dev` (and friends) to the Dockerfile if you do.
- **Other targets** — everything here generalizes: swap `--platform linux/arm64` and the Dockerfile's `FROM` line for the board/OS you're actually shipping to, and re-check which of libssl/libsrtp2/libusrsctp/libwebsockets that distro's packages satisfy before deciding what to build from source (see [Dependency requirements](../README.md#dependency-requirements)).
