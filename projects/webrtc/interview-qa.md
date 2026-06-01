# WebRTC 面试问答

## 基础架构

Q：WebRTC Native C++ 的最核心对象是什么？
A：`PeerConnectionFactoryInterface` 和 `PeerConnectionInterface`。前者创建 peer connection、audio/video track、codec 能力；后者负责 `AddTrack()`、`CreateOffer()`、`CreateAnswer()`、`SetLocalDescription()`、`SetRemoteDescription()`、`AddIceCandidate()`。源码入口在 `api/peer_connection_interface.h`。

Q：WebRTC 和普通 RTMP/播放器架构最大的差异是什么？
A：WebRTC 是交互式实时通信栈，核心是低延迟、双向、拥塞控制、NAT 穿透和端到端媒体协商；RTMP/播放器更多是推流/分发/缓冲播放模型。WebRTC 内部同时处理 ICE、DTLS-SRTP、RTP/RTCP、音视频编解码、抖动缓冲、丢包恢复、带宽估计。

Q：WebRTC 是否提供信令服务器？
A：不提供。WebRTC 生成 SDP 和 ICE candidate，但如何传给对端由业务信令决定。示例 `examples/peerconnection/client/conductor.cc` 用 JSON 传递 offer/answer/candidate，只是 demo。

## 通话流程

Q：主叫端发起通话的流程是什么？
A：创建 factory，创建 peer connection，添加 audio/video track，`CreateOffer()`，`SetLocalDescription()`，通过信令发送 offer，收集 `OnIceCandidate()` 并发送，收到 answer 后 `SetRemoteDescription()`，收到对端 candidate 后 `AddIceCandidate()`。官方流程写在 `api/peer_connection_interface.h:30` 附近。

Q：被叫端接收通话的流程是什么？
A：创建 factory 和 peer connection，收到 offer 后 `SetRemoteDescription()`，调用 `CreateAnswer()`，`SetLocalDescription(answer)`，通过信令返回 answer，同时交换 ICE candidates。官方流程写在 `api/peer_connection_interface.h:54` 附近。

Q：为什么添加 track 后还要重新协商？
A：`AddTrack()` 会改变媒体描述，需要通过 offer/answer 让对端知道新增的 m-line、方向、codec、ssrc/msid 等信息。`api/peer_connection_interface.h:803` 定义 `AddTrack()`，`api/peer_connection_interface.h:1006` 定义 `CreateOffer()`。

## 音频

Q：为什么 WebRTC 通话默认用 Opus？
A：Opus 同时适合语音和音乐，支持低延迟帧长、48kHz clock、动态码率、FEC、DTX、PLC，并且浏览器和 Native 互通成熟。WebRTC 的 NetEq、Audio Network Adaptor、voice engine 都围绕实时弱网做了适配。

Q：Opus 的 SDP 为什么通常是 `opus/48000/2`？
A：这是 RTP payload format 的 clock 和 channel capability。实际编码单声道还是立体声由 `stereo` 参数控制。`api/audio_codecs/opus/audio_decoder_opus.cc:41` 检查 `clockrate_hz == 48000` 且 `num_channels == 2`；`media/engine/webrtc_voice_engine.cc:1220` 说明只有 `stereo=1` 才编码双声道。

Q：WebRTC 能不能用 AAC？
A：默认浏览器互通通话不使用 AAC。Native 可以自定义 audio codec factory，但对端必须支持同样的 SDP、RTP payload、packetization、解码和 jitter 处理。实时通话优先 Opus。

## 视频

Q：WebRTC 默认支持哪些视频 codec？
A：默认软件路径通常有 VP8、VP9；H264 取决于 `WEBRTC_USE_H264`；AV1 encoder 取决于 `RTC_USE_LIBAOM_AV1_ENCODER`，AV1 decoder 取决于 `RTC_DAV1D_IN_INTERNAL_DECODER_FACTORY`。看 `media/engine/internal_encoder_factory.cc` 和 `media/engine/internal_decoder_factory.cc`。

Q：H265 是否是 WebRTC 默认通话能力？
A：不是。`api/video_codecs/sdp_video_format.h` 有 `H265()` 声明，但默认内置 encoder/decoder factory 不等于完整 H265 互通能力。生产支持 H265 通常要自定义 codec factory、SDP 能力和 RTP packetization，并确认对端支持。

Q：视频帧输入支持哪些格式？
A：`VideoFrameBuffer::Type` 支持 `kNative`、`kI420`、`kI420A`、`kI422`、`kI444`、`kI010`、`kI210`、`kI410`、`kNV12`。软件编码最稳是 I420；GPU texture 建议走 `kNative`。源码在 `api/video/video_frame_buffer.h:62`。

## OWT

Q：OWT 是 WebRTC 的一部分吗？
A：不是。OWT 是 Open WebRTC Toolkit，是基于 WebRTC 的会议/媒体服务器/SDK 生态，解决多人会议、SFU/MCU、录制、转码、媒体分发等问题。

Q：什么时候要考虑 OWT 或 SFU？
A：一对一通话不需要；多人会议、服务端录制、转码、混流、发布订阅、直播分发时需要。此时纯 P2P WebRTC 会遇到上行带宽、房间管理、媒体路由和服务端处理能力的问题。

## 工程排查

Q：C++ 接入后没有远端画面，先查什么？
A：先查信令是否完成 offer/answer；再查 ICE 是否 connected；再查 `OnAddTrack()` 是否触发；再查 renderer/sink 是否绑定；最后查 codec 协商和解码器 factory。示例回调在 `examples/peerconnection/client/conductor.cc:282`。

Q：通话能建立但没有声音，先查什么？
A：查 audio track 是否 `AddTrack()`，ADM 是否采集到 PCM，Opus 是否协商成功，远端是否收到 RTP，NetEq/decoder 是否输出，播放设备是否启动。示例 audio track 创建在 `examples/peerconnection/client/conductor.cc:511`。

Q：弱网卡顿时主要看哪些模块？
A：音频看 Opus FEC/DTX、NetEq、packet loss、jitter；视频看 bandwidth estimation、pacer、NACK/FEC、jitter buffer、encoder bitrate/framerate adaptation；网络看 ICE selected candidate pair、RTT、丢包和可用带宽。
