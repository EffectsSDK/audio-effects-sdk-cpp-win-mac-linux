![Effects SDK logo](Assets/Logo.png "Logo")

# Audio Effects SDK for Windows, macOS, and Linux

## Real-time AI-Powered Audio Noise Suppression SDK

Experience flawless audio with our real-time AI-powered noise suppression solution. Enjoy super easy integration, allowing you to enhance your application’s audio quality quickly and efficiently.  

**Perfect for**:

* **Video Conferencing**: Ensure crystal-clear communication without background distractions.
* **Live Streaming**: Deliver professional-grade audio for live broadcasts and streams.
* **Recording Applications**: Capture high-quality audio by eliminating unwanted noise.

This repository contains a C++ Audio Effects SDK version that you can integrate into your project or product.

## Documentation

[Documentation Website](https://effectssdk.ai/sdk/audio/desktop/docs/)

## Samples

Find desktop sample application source code here:
[Audio Effects SDK Desktop Sample](https://github.com/EffectsSDK/audio-effects-sdk-desktop-sample)

## Obtaining Effects SDK License

Register and get license token at our official [Developer Portal](https://effectssdk.ai/cp/registration).

## Technical Details

- SDK is available for Windows 10+ x64/x86 platforms;
  
## Getting Started

### Windows Integration

To add the sdk to your project:

* Link `<sdk_dir>\lib\AudioEffectsSDK.lib`
* Add `<sdk_dir>\include` to your include directories.
* Place `<sdk_dir>\bin\AudioEffectsSDK.dll` next to your .exe file or add its path to the DLL search paths.

<details>
  <summary>CMake integration sample</summary>

Assumed you set AUDIO_EFFECTS_SDK_DIR var (`-DAUDIO_EFFECTS_SDK_DIR=<path\to\sdk>`)
```cmake
add_library(audio_effects_sdk SHARED IMPORTED)
add_library(effects_sdk::AudioEffectsSDK ALIAS audio_effects_sdk)
file(TO_CMAKE_PATH ${AUDIO_EFFECTS_SDK_DIR} AUDIO_EFFECTS_SDK_DIR_CMAKE_PATH)
set_target_properties(
    audio_effects_sdk PROPERTIES
    IMPORTED_LOCATION ${AUDIO_EFFECTS_SDK_DIR_CMAKE_PATH}/bin/AudioEffectsSDK.dll
    IMPORTED_IMPLIB ${AUDIO_EFFECTS_SDK_DIR_CMAKE_PATH}/lib/AudioEffectsSDK.lib
)
target_include_directories(audio_effects_sdk INTERFACE ${AUDIO_EFFECTS_SDK_DIR_CMAKE_PATH}/include)

# ...

target_link_libraries(${YOUR_TARGET} PRIVATE effects_sdk::AudioEffectsSDK)

# Optional
# Copies AudioEffectsSDK.dll next to your binary
if(WIN32)
    add_custom_command(
        TARGET ${YOUR_TARGET} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_if_different 
            "$<TARGET_FILE:effects_sdk::AudioEffectsSDK>" 
            "$<TARGET_FILE_DIR:${YOUR_TARGET}>/$<TARGET_FILE_NAME:effects_sdk::AudioEffectsSDK>"
        VERBATIM
    )
endif()

```

</details>

### Usage 

The Audio Effects SDK entry point is the [createAudioEffectsSDKFactory](https://effectssdk.ai/sdk/audio/desktop/docs/sdk__factory_8h.html#a3bafe4d5fd7138d725193127fa621217) function, which creates an instance of [ISDKFactory](https://effectssdk.ai/sdk/audio/desktop/docs/classaudio__effects__sdk_1_1_i_s_d_k_factory.html).

1. Create an ISDKFactory instance using `createAudioEffectsSDKFactory()`;
2. Authorize the instance with method `ISDKFactory::auth()`;
3. Create an [IPipeline](https://effectssdk.ai/sdk/audio/desktop/docs/classaudio__effects__sdk_1_1_i_pipeline.html) using `ISDKFactory::createPipeline()`;
4. Enable noise suppression using `IPipeline::setNoiseSuppressionEnabled()`;
5. Process the audio stream using `IPipeline::process()`;

Example
```cpp
#include <audio_effects_sdk/sdk_factory.h>

bool initializeAudioEffectsSDK() {
    sdkFactory = createAudioEffectsSDKFactory();
    audio_effects_sdk::IAuthResult* authResult = sdkFactory->auth("<YOUR_CUSTOMER_ID>", nullptr, nullptr);
    bool authorized = authResult->status() == audio_effects_sdk::AuthStatus::active;
    authResult->release();
    if (!authorized) {
        sdkFactory->release();
        return false;
    }
    sdkPipeline = _sdkFactory->createPipeline(
	    format, // e.g. audio_effects_sdk::AudioFormatType::pcmFloat32 
	    sampleRate, 
	    channelCount, // Currently only 1 channel (mono) is supported
	    0
    );
    sdkPipeline->setNoiseSuppressionEnabled(true);
    return true;
}

void processAudio()
{
    sdkPipeline->process(
		inputFrames,
		inputFrameCount,
		outputFrames,
		outputFrameCount
	);
}

void releaseAudioEffectsSDK()
{
    sdkPipeline->release();
    sdkPipeline = nullptr;
    sdkFactory->waitUntilAsyncWorkFinished();
    sdkFactory->release();
    sdkFactory = nullptr; 
}

```

[IPipeline::process()](https://effectssdk.ai/sdk/audio/desktop/docs/classaudio__effects__sdk_1_1_i_pipeline.html#a969db05c03f85a5bcaa974afea72b877) can be used concurrently from two threads (producer & consumer).

Example
```cpp

// In producer thread (e.g. microphone callback)
void onPushAudio(const void* inputFrames, uint32_t inputFrameCount)
{
    sdkPipeline->process(
		inputFrames,
		inputFrameCount,
		nullptr,
		0
	);
}

// In consumer thread (e.g. speaker callback)
uint32_t onPullAudio(void* output, uint32_t outputFrameCount)
{
    // Returns written frame count into output (may be less than outputFrameCount)
    return sdkPipeline->process(
		nullptr,
		0,
		output,
		outputFrameCount
	);
}
```

### Memory management

SDK classes inherit the [IRelease](https://effectssdk.ai/sdk/audio/desktop/docs/classaudio__effects__sdk_1_1_i_release.html) interface with `IRelease::release` method. Use this method to release SDK-created object when it is no longer needed.