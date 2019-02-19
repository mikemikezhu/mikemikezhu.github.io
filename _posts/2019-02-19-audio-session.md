---
layout: post
title:  "Introduction to AVAudioSession in iOS"
categories: [Mobile Dev]
tag: [ios, AVAudioSession]
author: Mikemike Zhu
---

{:toc}

## Abstract

Audio session is working as an intermediary object for application system to communicate with the underlying audio hardware, which is widely used in the applications to play ringtone, ringback tone, audio/video call, as well as sound files. This technical document will mainly discuss the usage AVAudioSession, in order to provide clearer understanding of audio session, and also prevent the misusage of AVAudioSession during the software development.

<br/>

## Introduction of AVAudioSession

This section will mainly introduce the correct usage of AVAudioSession, by referring to Apple's official documentation.

In addition, in order to allow developers to use AVAudioSession properly, the following information will also be concisely discussed in this section:

- Recommended usage of each audio session
- Race condition when using audio session
- Comparison between each audio session

<br/>

### 1. Audio session category

AVAudioSession category is used to specify how the audio is intended to be used in the application, and provide a way for application developers to manipulate the audio behaviours of the system.

By default, the AVAudioSession category is "AVAudioSessionCategorySoloAmbient".

The comparison among different AVAudioSession categories is illustrated as follows:

#### AVAudioSessionCategorySoloAmbient (Default)

"AVAudioSessionCategorySoloAmbient" is the **default** AVAudioSession category.
In specific, when the device is locked, or switched to be silent, then the audio will also be silenced.

It is noteworthy that, by using such category, the audio is **non-mixable**, which means the activated audio session will interfere other audio sessions which are also non-mixable.
If developers need the audio to be mixable, "AVAudioSessionCategoryAmbient" shall be used in this case.

#### AVAudioSessionCategoryAmbient

"AVAudioSessionCategoryAmbient" is the AVAudioSession category which is used for non-primary sound playback.

In specific, when the device is locked, or switched to be silent, then the audio will also be silenced.

Additionally, the audio can be mixed with others by using such category.

**Usage:** used to play audio which is intended to be **mixed** with the audio from other apps, such as playing the music sound in the game application.

#### AVAudioSessionCategoryPlayback

"AVAudioSessionCategoryPlayback" is the AVAudioSession category which is used to play audio which needs to be played even though the device is locked or switched to be silent.

In other words, the audio will continue to play, on the condition that the device is locked, or switched to be silent.

By default, using this category implies that audio will be non-mixable.

If developers need the audio to be mixable, "AVAudioSessionCategoryOptionMixWithOthers" shall be set as category option.

**Usage:** used to play audio which needs to be played even though the device is locked or switched to be silent, such as **music player application**.

#### AVAudioSessionCategoryRecord

"AVAudioSessionCategoryRecord" is the AVAudioSession category which is used to record the audio while silencing other playback audio.

However, such category will not prevent the audio session from being interrupted by other non-mixable audio sessions, such as phone calls, alarms, etc.

In addition, permissions may need to be granted when using such category for audio recording.

**Usage:** used to record the audio which does not want to be interfered by other non-mixable playback audio sessions.

#### AVAudioSessionCategoryPlayAndRecord

"AVAudioSessionCategoryPlayAndRecord" is the AVAudioSession category which is used in the applications which need both recording and playing the audio.

Furthermore, the audio will continue to play, even though the device is locked, or switched to be silent.

By default, using this category implies that audio will be non-mixable.

If developers need the audio to be mixable, "AVAudioSessionCategoryOptionMixWithOthers" shall be set as category option.

**Usage:** used in the applications which need record audio and play audio feature, such as **making a phone call**.

#### AVAudioSessionCategoryMultiRoute

"AVAudioSessionCategoryMultiRoute" is the AVAudioSession category which is used to output audio data to various routes simultaneously.

If developers need the audio to be mixable, "AVAudioSessionCategoryOptionMixWithOthers" shall be set as category option.

**Usage:** used in the applications which need route audio to different output devices.

#### AVAudioSessionCategoryAudioProcessing (Deprecated)

"AVAudioSessionCategoryAudioProcessing" is the AVAudioSession category which is used to handle audio format conversion.

Specifically, while using an audio hardware codec or a signal processor, such category will automatically disable both audio output and audio input.

It is noteworthy that, such category has already been deprecated since iOS 10.0.

**Usage:** used in the applications which need offline audio format conversion.

<br/>

| **Audio session category** | **Silenced when device locked** | **Mixable** | **Usage** |
| AVAudioSessionCategorySoloAmbient (Default) | Yes | No | Play |
| AVAudioSessionCategoryAmbient | Yes | Yes | Play |
| AVAudioSessionCategoryPlayback | No | No by default (Yes by setting "AVAudioSessionCategoryOptionMixWithOthers") | Play |
| AVAudioSessionCategoryRecord | No | No | Record |
| AVAudioSessionCategoryPlayAndRecord | No | No by default (Yes by setting "AVAudioSessionCategoryOptionMixWithOthers") | Play / Record |
| AVAudioSessionCategoryMultiRoute | No | No by default (Yes by setting "AVAudioSessionCategoryOptionMixWithOthers") | Play / Record |
| AVAudioSessionCategoryAudioProcessing | - | - | Audio procession |

<br/>

### 2. Audio session category option

Additionally, category options are used to specify the behaviour of the current active audio session category.

The comparison between different AVAudioSession category options are described as follows:

#### AVAudioSessionCategoryOptionMixWithOthers

"AVAudioSessionCategoryOptionMixWithOthers" is the category option which allows the audio to be **mixable** with the audio from other applications.

**Race conditions:**

- This category option is only compatible with "AVAudioSessionCategoryPlayAndRecord", "AVAudioSessionCategoryPlayback", and "AVAudioSessionCategoryMultiRoute".

- If the category is set to be "AVAudioSessionCategoryAmbient", such category option is set automatically, which means the audio will be mixable.

- If the category option is set to be "AVAudioSessionCategoryOptionDuckOthers" and "AVAudioSessionCategoryOptionInterruptSpokenAudioAndMixWithOthe rs", such category option is set automatically.

#### AVAudioSessionCategoryOptionDuckOthers

"AVAudioSessionCategoryOptionDuckOthers" is the category option which allows the audio from other applications to be ducked (**reduced in volumn**), when the audio from the current session plays.

**Race conditions:**

- This category option is only compatible with "AVAudioSessionCategoryPlayAndRecord", "AVAudioSessionCategoryPlayback", and "AVAudioSessionCategoryMultiRoute".

- Setting this category option will also set "AVAudioSessionCategoryOptionMixWithOthers" category option.

#### AVAudioSessionCategoryOptionAllowBluetooth

"AVAudioSessionCategoryOptionAllowBluetooth" is the category option which allows audio input and output to a paired Bluetooth Hands-Free Profile (HFP) device.

**Race conditions:**

- This category option is only compatible with "AVAudioSessionCategoryPlayAndRecord", and "AVAudioSessionCategoryRecord".

#### AVAudioSessionCategoryOptionDefaultToSpeaker
"AVAudioSessionCategoryOptionDefaultToSpeaker" is the category option which specifies the default audio output to be the built-in speaker.

**Race conditions:**
- This category option is only compatible with "AVAudioSessionCategoryPlayAndRecord".

- By setting such category option, audio will **always route to the speaker**, rather than receiver, if no other accessory, such as headphones, are in use.

- **Changing audio session route will not reset this category option.** Only changing the audio session category will reset this category option.

#### AVAudioSessionCategoryOptionInterruptSpokenAudioAndMixWithOthers

"AVAudioSessionCategoryOptionInterruptSpokenAudioAndMixWithOthers" is the category option which will pause the continuous spoken content from other applications, when the audio from the current session plays.

**Race conditions:**

- This category option is only compatible with "AVAudioSessionCategoryPlayAndRecord", "AVAudioSessionCategoryPlayback", and "AVAudioSessionCategoryMultiRoute".

- Setting this category option will also set "AVAudioSessionCategoryOptionMixWithOthers" category option.

- If this category option is set, the current audio content will be mixed with other audio sessions.

- However, audio content will be paused if using the "AVAudioSessionModeSpokenAudio" audio session mode.

#### AVAudioSessionCategoryOptionAllowBluetoothA2DP

"AVAudioSessionCategoryOptionAllowBluetoothA2DP" is the category option which allows the audio session to be streamed to Bluetooth device which supports the Advanced Audio Distribution Profile (A2DP).

**Race conditions:**

- If using "AVAudioSessionCategoryAmbient", "AVAudioSessionCategorySoloAmbient", and "AVAudioSessionCategoryPlayback", then the system will automatically route to A2DP port.

- If using "AVAudioSessionCategoryPlayAndRecord", together with such category option, then the system will also route to A2DP port, since iOS 10.0.

- If using "AVAudioSessionCategoryMultiRoute" and "AVAudioSessionCategoryRecord", then this option will be cleared, and Bluetooth A2DP devices will not be available output routes.

#### AVAudioSessionCategoryOptionAllowAirPlay

"AVAudioSessionCategoryOptionAllowAirPlay" is the category option which allows the audio session to be streamed to AirPlay devices.

**Race conditions:**

- If using "AVAudioSessionCategoryPlayAndRecord", together with such category option, then the system will also route to AirPlay devices.

- If using "AVAudioSessionCategoryMultiRoute" and "AVAudioSessionCategoryRecord", then this option will be cleared, and AirPlay devices will not be available output routes.

<br/>

| **Audio session category option** | **Compatible with audio session category** | **Usage** |
| AVAudioSessionCategoryOptionMixWithOthers | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryPlayback AVAudioSessionCategoryMultiRoute | Allow audio to be **mixable** with the audio from other applications |
| AVAudioSessionCategoryOptionDuckOthers | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryPlayback AVAudioSessionCategoryMultiRoute | Allow the audio from other applications to **reduce the volumn**, when playing the current audio content |
| AVAudioSessionCategoryOptionAllowBluetooth | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryRecord | Bluetooth HFP device |
| AVAudioSessionCategoryOptionDefaultToSpeaker | AVAudioSessionCategoryPlayAndRecord | **Always route to speaker** |
| AVAudioSessionCategoryOption InterruptSpokenAudioAndMixWithOthers | AVAudioSessionCategoryPlayAndRecord AVAudioSessionCategoryPlayback AVAudioSessionCategoryMultiRoute | Allow audio to be mixable, but pause the spoken content |
| AVAudioSessionCategoryOptionAllowBluetoothA2DP | AVAudioSessionCategoryPlayAndRecord | Bluetooth A2DP device |
| AVAudioSessionCategoryOptionAllowAirPlay | AVAudioSessionCategoryPlayAndRecord | AirPlay device |

<br/>

### 3. Audio session mode

Never can we neglect the fact that, developers are able to use the audio session mode, together with the audio session category, in order to configure the system how the audio will be used in the applications.

The audio system will be configured for various use cases such as video chat or voice calling, video recording, etc.

By default, the AVAudioSession category is "AVAudioSessionModeDefault".

The comparison between different AVAudioSession modes are demonstrated below:

#### AVAudioSessionModeDefault (Default)

"AVAudioSessionModeDefault" is the **default** AVAudioSession mode, which is compatible with every audio session category.

#### AVAudioSessionModeMoviePlayback

"AVAudioSessionModeMoviePlayback" is used to enhance the signal processing of **movie playback** for certain audio routes such as built-in speaker or headphones.
This mode is only compatible with “AVAudioSessionCategoryPlayback” audio session category.

#### AVAudioSessionModeVideoRecording

"AVAudioSessionModeVideoRecording" is used to **record a video**. Specifically, if the device has more than one built-in microphones, the audio session will use the microphones which is closet to the video camera.

**Race conditions:**

This mode is only compatible with “AVAudioSessionCategoryPlayAndRecord" and "AVAudioSessionCategoryRecord" audio session category.

#### AVAudioSessionModeVoiceChat

"AVAudioSessionModeVoiceChat" is used in applications with features such as VoIP **voice call** in order to perform two-way communication.

**Race conditions:**

This mode is only compatible with “AVAudioSessionCategoryPlayAndRecord” audio session category.

#### AVAudioSessionModeGameChat

"AVAudioSessionModeGameChat" is used in Game Kit applications for **game voice chat** service.

**Race conditions:**

This mode is only compatible with “AVAudioSessionCategoryPlayAndRecord” audio session category.
 
#### AVAudioSessionModeVideoChat
"AVAudioSessionModeVideoChat" is used in applications which need **video call**, such as Facetime, Skype or WeChat video call, etc.

In specific, the tonal equalization as well as audio routes will be optimized for video call, by using such mode.

**Race conditions:**
This mode is only compatible with “AVAudioSessionCategoryPlayAndRecord” and "AVAudioSessionCategoryRecord" audio session category.

#### AVAudioSessionModeSpokenAudio

"AVAudioSessionModeSpokenAudio" is used in the scenario that user wants to pause the audio playback, and starts to play the continuous spoken audio. 

**Race conditions:**

This mode is only compatible with “AVAudioSessionCategoryPlayback” audio session category.

#### AVAudioSessionModeMeasurement

"AVAudioSessionModeMeasurement" is used when performing the measurement of audio input / output.

Specifically, it will reduce the amount of the system audio input / output to the minimum, for the purpose of performing measurement of the application's audio.

**Race conditions:**
This mode is only compatible with “AVAudioSessionCategoryPlayback”, "AVAudioSessionCategoryRecord", "AVAudioSessionCategoryPlayAndRecord" audio session category.

<br/>

| **Audio session mode** | **Compatible with audio session category** | **Usage** |
| AVAudioSessionModeDefault | All | Default |
| AVAudioSessionModeMoviePlayback | AVAudioSessionCategoryPlayback | Movie playback |
| AVAudioSessionModeVideoRecording | AVAudioSessionCategoryRecord AVAudioSessionCategoryPlayAndRecord | Video recording |
| AVAudioSessionModeVoiceChat | AVAudioSessionCategoryPlayAndRecord | Voice call |
| AVAudioSessionModeGameChat | AVAudioSessionCategoryPlayAndRecord | Game chat |
| AVAudioSessionModeVideoChat | AVAudioSessionCategoryRecord AVAudioSessionCategoryPlayAndRecord | Video call |
| AVAudioSessionModeSpokenAudio | AVAudioSessionCategoryPlayback | Spoken audio |
| AVAudioSessionModeMeasurement | AVAudioSessionCategoryPlayback AVAudioSessionCategoryRecord AVAudioSessionCategoryPlayAndRecord | Measurement of audio input / output |

<br/>

### 4. Port types

The eligible *output* port types are illustrated as follows:

#### AVAudioSessionPortLineOut
"AVAudioSessionPortLineOut" refers to the line-level output of the dock connector.

#### AVAudioSessionPortHeadphones
"AVAudioSessionPortHeadphones" refers to the head phones or wired header set devices.

#### AVAudioSessionPortBluetoothA2DP
"AVAudioSessionPortBluetoothA2DP" refers to the bluetooth A2DP device.

#### AVAudioSessionPortBuiltInReceiver
"AVAudioSessionPortBuiltInReceiver" refers to the **built-in receiver** of the device.

#### AVAudioSessionPortBuiltInSpeaker
"AVAudioSessionPortBuiltInSpeaker" refers to the **built-in speaker** of the device.

#### AVAudioSessionPortHDMI
"AVAudioSessionPortHDMI" refers to the output via HDMI specification.

#### AVAudioSessionPortAirPlay
"AVAudioSessionPortAirPlay" refers to the AirPlay device.

#### AVAudioSessionPortBluetoothLE
"AVAudioSessionPortBuetoothLE" refers to the Bluetooth Low Energy device.

The eligible both *input* and *output* port types are illustrated as follows:

#### AVAudioSessionPortBluetoothHFP
"AVAudioSessionPortBuetoothHFP" refers to the Bluetooth HFP device.

#### AVAudioSessionPortUSBAudio
"AVAudioSessionPortUSBAudio" refers to the USB device.

#### AVAudioSessionPortCarAudio
"AVAudioSessionPortCarAudio" refers to the car audio input / output.

### 5. Change audio session route

Furthermore, audio session route can be **temporarily** changed by using the following API:

```c
- (BOOL)overrideOutputAudioPort:(AVAudioSessionPortOverride)portOverride 
                          error:(NSError * _Nullable *)outError;
```

Developers are able to override with the following options: 
- AVAudioSessionPortOverrideNone
- AVAudioSessionPortOverrideSpeaker

<br/>

### 6. Observing audio session route changes

Last but not least, the audio session route changes could be observed by observing "AVAudioSessionRouteChangeNotification".

In specific, developers are able to know the previous route by checking "AVAudioSessionRouteChangePreviousRouteKey" in the route change notification user info.

<br/>

## References

1. [AVAudioSession](https://developer.apple.com/documentation/avfoundation/ avaudiosession?language=objc)
2. [AVAudioSession categories](https://developer.apple.com/documentation/avfoundation/ avaudiosession/audio_session_categories?language=objc)
3. [AVAudioSession modes](https://developer.apple.com/documentation/avfoundation/ avaudiosession/audio_session_modes?language=objc)
4. [Override ouput audio port](https://developer.apple.com/documentation/avfoundation/ avaudiosession/1616443-overrideoutputaudioport?language=objc)
5. [AVAudioSession route change notification](https://developer.apple.com/documentation/avfoundation/ avaudiosessionroutechangenotification)