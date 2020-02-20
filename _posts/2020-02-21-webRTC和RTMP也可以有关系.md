---
layout:     post
title:      webRTC和RTMP也能有关系
subtitle:   webRTC to RTMP, RTMP to webRTC
date:       2020-02-21
author:     Sunw
header-img: img/post-bg-unix-linux.jpg
catalog: true
tags:
    - RTMP
    - 直播
    - webRTC
    - 会议
    - gstreamer
---

本文基于gstreamer开源库,进行RTMP流转RTP流，送到webRTC播放;webRTC视频会议发送过来的RTP流，转会为RTMP流，可以推送到直播服务器。支持opus,AAC,h264,vp8。

## RTMP->webRTC

录制flv文件
```
gst-launch-1.0 -v  -e  rtmpsrc location=rtmp://127.0.0.1:1935/live/stream1 ! filesink location= 123.flv
```

---

### 视频
录制mpg文件
```
gst-launch-1.0 -v -m rtmpsrc location=rtmp://127.0.0.1:1935/live/stream1 ! flvdemux ! h264parse config-interval=-1 ! mpegtsmux ! filesink location= 123.mpg
```
录制yuv
```
gst-launch-1.0 -v -m rtmpsrc location=rtmp://127.0.0.1:1935/live/stream1 ! flvdemux ! h264parse config-interval=-1 ! avdec_h264 ! filesink location= 1234.yuv
```

录制vp8
```
gst-launch-1.0 -v -m rtmpsrc location=rtmp://127.0.0.1:1935/live/stream1 ! flvdemux ! h264parse config-interval=-1 ! avdec_h264 ! vp8enc ! webmmux ! filesink location= 1234.webm
```


##### RTMP转封装RTP
此部分使用了gstreamer, 只所以用gstreamer是因为发现ffmpeg的转出来的rtp包, 有一定概率webrtc会解析失败, 还未找到具体原因

RTMP转封装成RTP(h264)
```
gst-launch-1.0 -v  rtmpsrc location=rtmp://localhost/live/{stream} ! flvdemux ! h264parse ! rtph264pay config-interval=-1 pt={pt} !  udpsink host=127.0.0.1 port={port}

gst-launch-1.0 -v  rtmpsrc location=rtmp://127.0.0.1:1935/live/stream1 ! flvdemux ! h264parse ! rtph264pay config-interval=-1 pt=96 ! filesink location= 123.flv
```
##### RTMP转封装成RTP(vp8)
```
gst-launch-1.0 -v -m rtmpsrc location=rtmp://127.0.0.1:1935/live/stream1 ! flvdemux ! h264parse config-interval=-1 ! avdec_h264 ! vp8enc ! rtpvp8pay pt=96 ! udpsink host=127.0.0.1 port=11111
```

---

### 声音

##### RTMP转封装成RTP(opus)
```
gst-launch-1.0 -v -m  uridecodebin uri=rtmp://127.0.0.1:1935/live/stream1 ! audioconvert ! audioresample ! opusenc ! rtpopuspay timestamp-offset=0 pt=111 ! udpsink host=127.0.0.1 port=11111

```

## webRTC->RTMP

生成flv文件
```
gst-launch-1.0 -v flvmux name=mux ! filesink location=test.flv  audiotestsrc samplesperbuffer=44100 num-buffers=10 ! faac ! mux.  videotestsrc num-buffers=250 ! video/x-raw,framerate=25/1 ! x264enc ! mux.
```
推流(测试数据)
```
gst-launch-1.0 -v flvmux name=mux ! rtmpsink location="rtmp://127.0.0.1:1935/live/stream1" audiotestsrc samplesperbuffer=44100 num-buffers=10 ! faac ! mux.  videotestsrc num-buffers=250 ! video/x-raw,framerate=25/1 ! x264enc ! mux.
```

从webRTC接收音频和视频，opus转码为AAC,将AAC和H264 封装为FLV格式，推流到RTMP服务
```
gst-launch-1.0 -v flvmux name=mux ! rtmpsink location="rtmp://127.0.0.1:1935/live/stream1" udpsrc port=28518 caps="application/x-rtp, media=audio, encoding-name=OPUS, clock-rate=48000" ! rtpopusdepay ! opusdec ! faac ! mux.  udpsrc port=28519 caps="application/x-rtp, media=video, encoding-name=H264, clock-rate=90000" ! rtph264depay ! mux.
```


