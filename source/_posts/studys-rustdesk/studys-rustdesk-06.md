---
title: RustDesk源码学习笔记 06-分析客户端通信报文1
date: 2025-03-02 18:18:00
updated:
tags:
  - "文章"
  - "学习笔记"
categories:
  - "学习笔记"
  - "RustDesk源码学习笔记"
description: "学习一下RustDesk开源项目，试图了解udp，tokio，p2p,rdp的一些知识"
cover: https://jsd.012700.xyz/gh/jerryc127/CDN@latest/cover/default_bg.png
---

## 介绍
### 概要
RustDesk客户端之间通过protobuf定义的报文格式进行数据传输
在这些数据传输的报文有26种格式，其中包括：
SignedId
PublicKey
TestDelay
VideoFrame
LoginRequest
LoginResponse
Hash
MouseEvent
AudioFrame
CursorData
CursorPosition
uint64
KeyEvent
Clipboard
FileAction
FileResponse
Misc
Cliprdr
MessageBox
SwitchSidesResponse
VoiceCallRequest
VoiceCallResponse
PeerInfo
PointerDeviceEvent
Auth2FA
MultiClipboards

## 正文
### SignedId和PublicKey
#### 介绍
当主控端和被控端建立连接后,由被控端先发起SignedId报文,主控端将返回PublicKey.
这个过程中,根据情况主控端和被控端建立加密通信或非加密通信.

### TestDelay
#### 报文定义
```proto
message TestDelay {
  int64 time = 1;
  bool from_client = 2;
  uint32 last_delay = 3;
  uint32 target_bitrate = 4;
}
```
#### 介绍
被控端每隔1秒发送给主控端一次，主控端会直接将请求转发回被控端，被控端收到后，会记录发送和接受之间的间隔作为延迟。
TestDelay.last_delay 延迟【上一轮TestDelay请求，被控端发送到响应的间隔】
TestDelay.target_bitrate 码率
TestDelay.from_client 请求来自主控端还是被控端。请求总是来自于被控端，总为false

### VideoFrame
#### 报文定义
```proto
message EncodedVideoFrame {
  bytes data = 1;
  bool key = 2;
  int64 pts = 3;
}

message EncodedVideoFrames { repeated EncodedVideoFrame frames = 1; }

message RGB { bool compress = 1; }

// planes data send directly in binary for better use arraybuffer on web
message YUV {
  bool compress = 1;
  int32 stride = 2;
}

message VideoFrame {
  oneof union {
    EncodedVideoFrames vp9s = 6;
    RGB rgb = 7;
    YUV yuv = 8;
    EncodedVideoFrames h264s = 10;
    EncodedVideoFrames h265s = 11;
    EncodedVideoFrames vp8s = 12;
    EncodedVideoFrames av1s = 13;
  }
  int32 display = 14;
}

```
#### 介绍
视频帧，可用于传输多种编码格式的视频帧
主控端收到视频帧后，会将其交由队列进行执行
队列中传递以下类型的数据
```rust
/// Media data.
pub enum MediaData {
    VideoQueue,
    VideoFrame(Box<VideoFrame>),
    AudioFrame(Box<AudioFrame>),
    AudioFormat(AudioFormat),
    Reset,
    RecordScreen(bool),
}
```

### LoginRequest和LoginResponse
#### 介绍
LoginRequest用于主控端请求登陆被控端，只有通过认证后，主控端才能控制被控端。
如果认证失败，被控端通过LoginResponse告知主控端认证失败的理由。
如果认证成功，被控端通过LoginResponse传递以下信息给主控端。
```proto
message PeerInfo {
  string username = 1;
  string hostname = 2;
  string platform = 3;
  repeated DisplayInfo displays = 4;
  int32 current_display = 5;
  bool sas_enabled = 6;
  string version = 7;
  Features features = 9;
  SupportedEncoding encoding = 10;
  SupportedResolutions resolutions = 11;
  // Use JSON's key-value format which is friendly for peer to handle.
  // NOTE: Only support one-level dictionaries (for peer to update), and the key is of type string.
  string platform_additions = 12;
  WindowsSessions windows_sessions = 13;
}
```

### Hash
#### 介绍
当主控端和被控端建立连接后，首先完成SignedId和PublicKey的认证过程。
然后被控端会发送Hash报文到主控端，主控端收到报文后会开始发送LoginRequest的过程
```rust
let hash = Hash {
    salt: Config::get_salt(), //6位随机字符
    challenge: Config::get_auto_password(6), //6位随机字符
    ..Default::default()
};
```

### MouseEvent
#### 介绍
平平无奇的鼠标事件，使用enigo跨平台模拟输入库实现。

### AudioFrame
#### 介绍
平平无奇的声音帧，注意：主控端和被控端均可以接受来自对方的声音帧

### CursorData
#### 介绍
主控端向被控端传输光标位置

### CursorId和CursorPosition
#### 介绍
CursorId没找到具体代码
CursorPosition和光标位置有关，可能还和切换屏幕有关

### KeyEvent
#### 介绍
平平无奇的按键事件

### Clipboard、MultiClipboards
#### 介绍
平平无奇的剪切板，使用arboard库处理跨平台剪切板。
Clipboard只支持复制文本，MultiClipboards支持复制多种格式。
```proto
enum ClipboardFormat {
  Text = 0;
  Rtf = 1;
  Html = 2;
  ImageRgba = 21;
  ImagePng = 22;
  ImageSvg = 23;
  Special = 31;
}

message Clipboard {
  bool compress = 1;
  bytes content = 2;
  int32 width = 3;
  int32 height = 4;
  ClipboardFormat format = 5;
  // Special format name, only used when format is Special.
  string special_name = 6;
}

message MultiClipboards { repeated Clipboard clipboards = 1; }
```



### Cliprdr
#### 介绍
看上去是传输剪切板文件和其他需要延迟生成的剪切板信息

### FileAction
#### 介绍
负责指示文件动作，如远程ls，远程传输文件之类的命令

### FileResponse
#### 介绍
实际传输文件，以及传输文件的结果

### Misc
#### 介绍
杂项信息，传输多种种类的信息
```proto

message Misc {
  oneof union {
    ChatMessage chat_message = 4;
    SwitchDisplay switch_display = 5;
    PermissionInfo permission_info = 6;
    OptionMessage option = 7;
    AudioFormat audio_format = 8;
    string close_reason = 9;
    bool refresh_video = 10;
    bool video_received = 12;
    BackNotification back_notification = 13;
    bool restart_remote_device = 14;
    bool uac = 15;
    bool foreground_window_elevated = 16;
    bool stop_service = 17;
    ElevationRequest elevation_request = 18;
    string elevation_response = 19;
    bool portable_service_running = 20;
    SwitchSidesRequest switch_sides_request = 21;
    SwitchBack switch_back = 22;
    // Deprecated since 1.2.4, use `change_display_resolution` (36) instead.
    // But we must keep it for compatibility when peer version < 1.2.4.
    Resolution change_resolution = 24;
    PluginRequest plugin_request = 25;
    PluginFailure plugin_failure = 26;
    uint32 full_speed_fps = 27; // deprecated
    uint32 auto_adjust_fps = 28;
    bool client_record_status = 29;
    CaptureDisplays capture_displays = 30;
    int32 refresh_video_display = 31;
    ToggleVirtualDisplay toggle_virtual_display = 32;
    TogglePrivacyMode toggle_privacy_mode = 33;
    SupportedEncoding supported_encoding = 34;
    uint32 selected_sid = 35;
    DisplayResolution change_display_resolution = 36;
    MessageQuery message_query = 37;
    int32 follow_current_display = 38;
  }
}
```

### MessageBox
#### 介绍
被控端向主控端发送弹窗指令

### Misc.SwitchSidesRequest和SwitchSidesResponse
#### 介绍
主控端A主动进行换边处理，并发送SwitchSidesRequest至被控端B。
被控端B收到SwitchSidesRequest后，主动连接主控端A，并关闭远程连接。
建立连接后，B端作为主控端，A端作为被控端。
B端将向A端发送SwitchSidesResponse请求，A端收到后即完成校验，免去LoginRequest和LoginResponse的步骤。

### VoiceCallRequest和VoiceCallResponse
#### 介绍
请求通话和请求通话的响应

### PeerInfo
#### 介绍
登陆报文中包含PeerInfo，单独的PeerInfo同样返回端点的信息（显示器信息，平台信息等）


### PointerDeviceEvent
#### 介绍
传输触摸输入设备的信息

### Auth2FA
#### 介绍
和认证相关，可能是验证码
