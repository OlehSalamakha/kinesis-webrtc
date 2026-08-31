# Building and Running Samples with IoT Certificates

This guide covers building the samples with AWS IoT Core certificate-based
credentials instead of static AWS access keys, and how to run them once built.

See also the main [Setup IoT](../README.md#setup-iot) section in the README.

## Prerequisites

You need three files per device/thing, typically obtained when provisioning
the device in AWS IoT Core:

* Device certificate (`<id>-certificate.pem.crt`)
* Device private key (`<id>-private.pem.key`)
* Amazon Root CA (`AmazonRootCA1.pem`)

You'll also need, from your IoT setup:

* IoT credentials endpoint (`<prefix>.credentials.iot.<region>.amazonaws.com`)
* IoT role alias name
* IoT thing name (device ID)

## Build

Configure the build with `-DIOT_CORE_ENABLE_CREDENTIALS=ON`:

```shell
mkdir -p kvs-webrtc-sdk/build && cd kvs-webrtc-sdk/build
cmake .. -DBUILD_SAMPLES=ON -DIOT_CORE_ENABLE_CREDENTIALS=ON -DPARALLEL_BUILD=ON
make -j
```

Look for this line in the CMake output to confirm it's enabled:

```
Use IoT credentials in the samples
```

> [!NOTE]
> Building with `-DIOT_CORE_ENABLE_CREDENTIALS=ON` makes the samples ignore
> `AWS_ACCESS_KEY_ID` / `AWS_SECRET_ACCESS_KEY` at runtime in favor of the
> `AWS_IOT_CORE_*` environment variables below. If you need to switch back to
> static credentials later, reconfigure without this flag and rebuild.

## Run

Export the IoT credential environment variables before running any sample:

```bash
export AWS_IOT_CORE_CREDENTIAL_ENDPOINT=<prefix>.credentials.iot.<region>.amazonaws.com
export AWS_IOT_CORE_CERT=/path/to/certificates/<id>-certificate.pem.crt
export AWS_IOT_CORE_PRIVATE_KEY=/path/to/certificates/<id>-private.pem.key
export AWS_IOT_CORE_ROLE_ALIAS=<your-role-alias-name>
export AWS_IOT_CORE_THING_NAME=<your-thing-name>
export AWS_DEFAULT_REGION=<region>
```

Optionally, if the default trust bundle doesn't validate your IoT endpoint's
chain, point at the Amazon root CA explicitly:

```bash
export AWS_KVS_CACERT_PATH=/path/to/certificates/AmazonRootCA1.pem
```

Then, from the `build/` directory, run any sample exactly as documented in
the main README — no command-line syntax changes, only the environment
differs:

**P2P master (test frames):**
```bash
./samples/kvsWebrtcClientMaster <channelName> [mediaType] [audio-codec] [video-codec]
```

**P2P master via GStreamer:**
```bash
./samples/kvsWebrtcClientMasterGstSample <channelName> <mediaType> <sourceType> [audio-codec] [video-codec]
# sourceType: testsrc | devicesrc | rtspsrc <rtsp://...>
```

**P2P viewer:**
```bash
./samples/kvsWebrtcClientViewer <channelName> [audio-codec] [video-codec]
./samples/kvsWebrtcClientViewerGstSample <channelName> <mediaType> [audio-codec] [video-codec]
```

**WebRTC Storage (ingest) master:**
```bash
./samples/kvsWebrtcStorageAudioVideoMaster <channelName>
./samples/kvsWebrtcStorageVideoOnlyMaster <channelName>
./samples/kvsWebrtcStorageAudioVideoMasterGstSample <channelName> <srcType> [rtspUrl]
./samples/kvsWebrtcStorageVideoOnlyMasterGstSample <channelName> <srcType> [rtspUrl]
```

`<channelName>` is any name you choose — the sample creates the signaling
channel automatically if it doesn't already exist.

## Verifying the credential exchange

On a successful run, the logs show the IoT credentials endpoint returning
`200` and issuing temporary STS credentials, followed by KVS API calls
(`describeSignalingChannel`, `createSignalingChannel`, etc.) signed with
those credentials:

```
DEBUG   lwsIotCallbackRoutine(): Connected with server response: 200
DEBUG   parseIotResponse(): [<thingName>] Iot credential expiration time <epoch>
...
DEBUG   setRequestHeader(): Appending header to request: Authorization -> AWS4-HMAC-SHA256 Credential=ASIA.../.../kinesisvideo/aws4_request, ...
```

If instead you see `AWS_IOT_CORE_CREDENTIAL_ENDPOINT must be set` (or
similarly for `AWS_IOT_CORE_CERT`, `AWS_IOT_CORE_PRIVATE_KEY`,
`AWS_IOT_CORE_ROLE_ALIAS`, `AWS_IOT_CORE_THING_NAME`), double check the
corresponding environment variable is exported and non-empty.
