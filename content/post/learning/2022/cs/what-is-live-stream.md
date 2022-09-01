---
title: 什么是云直播
description: 云直播服务商的竞争点是精细化的场景划分；那么，人呢？
date: 2022-02-21
slug: what-is-live-stream
image: img/2022/02/MaldivesHeart.jpg
categories: [Learning]
tags: [Learning, CS, LiveStream]
---

## [Wiki](https://en.wikipedia.org/wiki/Livestreaming) Definition

Live Streaming is streaming media simultaneously recorded and broadcast in real-time over the Internet. It is often referred to simply as streaming, but this abbreviated term is ambiguous because "streaming" may refer to any media delivered and played back simultaneously without requiring a completely downloaded file.

## 简而言之

直播就是采集主播端的音视频，推送到云上的统一处理模块，然后经过边缘 SRS 集群、CDN 等工具分发给各地的观众端。

## 最简单的直播架构

![.](img/2022/02/simple-live-arch.svg)

## 国内知名服务提供商

### 腾讯云的直播架构

云直播（Cloud Streaming Services）为您提供极速、稳定、专业的直播云端处理服务，根据业务中不同直播场景的需求，云直播提供标准直播、快直播、慢直播和云导播台服务，分别针对大规模实时观看、高并发推流录制及超低延时的直播场景，配合移动直播 SDK，为您提供一站式的音视频直播解决方案。

![.](img/2022/02/tx-live-arch.png)

### 阿里云的直播架构

视频直播服务（ApsaraVideo Live）是基于领先的内容接入与分发网络和大规模分布式实时转码技术打造的音视频直播平台，提供便捷接入、高清流畅、低延迟、高并发的音视频直播服务。

![.](img/2022/02/ali-live-arch.png)

## 我们可以从中学到什么

对比上面几个直播架构后，我们会发现其中的思想基本是一致的，也就是说直播这套系统上玩不出什么太多新花样；于是更多的是从场景上寻找区分度，比如各家服务商都推出了好几种直播：快直播、慢直播、极速直播等等。那么，区别在哪里？

- **快直播**：基于 RTC 协议的直播，延迟在一秒以内
- **普通直播**：基于 RTMP 协议的直播，延迟一到三秒
- **慢直播**：适用于物联网场景，延迟很高

在架构体系已经接近固化的情况下，为“消费者”提供更多“应用”是一个不错的方向。

## 总结

我还是不明白直播 PK 为什么会成为热门应用场景。

## Further Learning

### 腾讯云音视频 SDK 设计架构

![.](img/2022/02/tx-live-sdk.svg)

### 腾讯云小直播解决方案

![.](img/2022/02/tx-lil-live.svg)

## Reference

- [腾讯云直播文档](https://cloud.tencent.com/document/product/267)
- [阿里云直播文档](https://help.aliyun.com/product/29949.html)
