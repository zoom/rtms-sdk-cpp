# RTMS SDK for C++

The RTMS (Real-Time Media Streaming) C++ SDK enables third-party app servers to connect to the RTMS gateway and retrieve video, audio, and transcript data from Zoom meetings in real-time.

## Supported Platforms

- Linux
- macOS

## Installation

### Linux

The SDK package for Linux is provided as a dynamic library (`.so` file).

**Contents:**
- Header files:
  - `rtms_common.h`
  - `rtms_sdk.h` (include this in your C++ source or header files)
  - `rtms_csdk.h` (for C interface - ignore when using C++ API)
- Library file: `librtmsdk.so.0`

### macOS

The SDK package for macOS is provided as a dylib file.

**Contents:**
- Header files:
  - `rtms_common.h`
  - `rtms_sdk.h` (include this in your C++ source or header files)
  - `rtms_csdk.h` (for C interface - ignore when using C++ API)
- Library files:
  - `tp.framework`
  - `util.framework`
  - `libzContext.dylib`
  - `libssl.dylib`
  - `librtms_sdk.dylib`
  - `libjson.dylib`
  - `libcrypto.dylib`
  - `curl64.framework`

## Features

The SDK provides access to the following data types from Zoom meetings:

- **Participant information** - Details about meeting participants
- **RTMS session information** - Session metadata and status
- **Audio data** - Real-time audio streams
- **Video data** - Real-time video streams
- **Desktop share data** - Screen sharing content
- **Transcript data** - Live meeting transcription
- **Chat data** - Meeting chat messages

## Usage

### 1. Create a Sink Object

Inherit from `rtms_sdk_sink` and implement the virtual callback functions:

```cpp
class sample_sink : public rtms_sdk_sink
{
public:
    sample_sink() {}
    ~sample_sink() {}

    virtual void on_join_confirm(int reason);
    virtual void on_session_update(int op, struct session_info *sess);
    virtual void on_user_update(int op, struct participant_info *pi);
    virtual void on_audio_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md);
    virtual void on_video_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md);
    virtual void on_ds_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md);
    virtual void on_transcript_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md);
    virtual void on_leave(int reason);
};
```

### 2. Initialize the SDK Provider

Initialize the `rtms_sdk_provider` singleton with a Certificate Authority (CA) file:

```cpp
rtms_sdk_provider::instance()->init("./cert/ca.crt");
```

### 3. Create and Configure SDK Instance

```cpp
// Create sink instance
sample_sink *_sink = new sample_sink;

// Create SDK instance
rtms_sdk *sdk = rtms_sdk_provider::instance()->create_sdk();

// Configure SDK
sdk->config(NULL, SDK_ALL, 0);
```

**Media Type Options:**
- `SDK_ALL` - All supported media types
- `SDK_AUDIO` - Audio only
- `SDK_VIDEO` - Video only
- `SDK_DESKSHARE` - Desktop sharing only
- `SDK_TRANSCRIPT` - Transcript only
- `SDK_CHAT` - Chat only

### 4. Join Meeting Session

```cpp
sdk->open(_sink);
sdk->join(<meeting_uuid>, <rtms_stream_id>, <signature>, <server_url>, 10000);
```

**Parameters:**
- `meeting_uuid` - Zoom meeting UUID (provided via webhook)
- `rtms_stream_id` - RTMS stream identity (provided via webhook)
- `signature` - HMACSHA256(client_id + "," + meeting_uuid + "," + rtms_stream_id, secret)
- `server_url` - RTMS gateway URL (provided via webhook)
- `timeout` - Join timeout in milliseconds

### 5. Main Loop

Keep the client running to receive media data continuously:

```cpp
while (running) {
    sdk->poll();
}
```

### 6. Cleanup

Release resources when done:

```cpp
rtms_sdk_provider::instance()->release_sdk(sdk);
```

## Complete Example

```cpp
#include "string.h"
#include "rtms_sdk.h"

bool s_is_running = true;

class sample_sink : public rtms_sdk_sink
{
public:
    sample_sink() {}
    ~sample_sink() {}

    virtual void on_join_confirm(int reason) {
        printf("on_join_confirm, reason: %d\n", reason);
    }

    virtual void on_session_update(int op, struct session_info *sess) {
        printf("on_session_update\n");
    }

    virtual void on_user_update(int op, struct participant_info *pi) {
        printf("on_user_update\n");
    }

    virtual void on_audio_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md) {
        printf("on_audio_data, timestamp:%lu\n", timestamp);
    }

    virtual void on_video_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md) {
        // Handle video data
    }

    virtual void on_ds_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md) {
        // Handle desktop share data
    }

    virtual void on_transcript_data(unsigned char *data_buf, int size, uint64_t timestamp, struct rtms_metadata *md) {
        printf("[%s]: %s\n", md->user_name, data_buf);
    }

    virtual void on_leave(int reason) {
        printf("on_leave\n");
        s_is_running = false;
    }
};

int main(int argc, char **argv)
{
    if (argc < 5) {
        printf("Usage: %s <meeting_uuid> <rtms_stream_id> <signature> <signal_url>\n", argv[0]);
        return -1;
    }

    std::string g_meeting_uuid = argv[1];
    std::string g_rtms_stream_id = argv[2];
    std::string g_signature = argv[3];
    std::string g_signal_url = argv[4];

    // Initialize SDK provider
    rtms_sdk_provider::instance()->init("./cert/ca.crt");

    // Create sink and SDK instances
    sample_sink *_sink = new sample_sink;
    rtms_sdk *sdk = rtms_sdk_provider::instance()->create_sdk();
    
    // Configure for all media types
    sdk->config(NULL, SDK_ALL, 0);
    
    // Open connection and join meeting
    sdk->open(_sink);
    sdk->join(g_meeting_uuid.c_str(), g_rtms_stream_id.c_str(), 
              g_signature.c_str(), g_signal_url.c_str(), 10000);

    // Main loop
    while (s_is_running) {
        sdk->poll();
    }

    // Cleanup
    rtms_sdk_provider::instance()->release_sdk(sdk);
    return 0;
}
```

## Building Your Application

When building your RTMS-enabled client, make sure to:

1. Include the header files in your source code
2. Link against the provided libraries
3. Ensure the CA certificate file is accessible at runtime

## Requirements

- C++11 or later
- Certificate Authority file for TLS verification
- Valid Zoom meeting credentials and webhook parameters

## Support

For questions and support, please refer to the Zoom Developer Documentation or contact Zoom Developer Support.
