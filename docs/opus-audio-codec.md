# OPUS 音频编码格式说明

## 什么是 OPUS？

**OPUS (Opus Interactive Audio Codec)** 是一种音频编码格式，于 2012 年发布。

### 基本信息
- **标准化组织**：IETF（RFC 6716）
- **授权方式**：开源、免费、无专利费用
- **主要用途**：实时通信（视频会议、语音通话）、网络直播
- **发布时间**：2012 年

## 技术特点

### 优点
- **压缩效率高** - 在低码率下音质优于 MP3、AAC
- **延迟低** - 适合实时通信场景
- **开源免费** - 无需支付专利授权费
- **灵活性强** - 支持多种码率和应用场景

### 缺点
- **兼容性差** - 大部分家庭影院设备、电视、老式播放器不支持
- **不是主流** - 在影视资源中很少使用
- **需要特定播放器** - 如 VLC、MPC-HC 等

## 为什么在过滤规则中排除 OPUS？

### 1. 兼容性问题
- 大部分家庭影院设备不支持 OPUS 解码
- 电视盒子、智能电视可能无法播放
- 某些媒体服务器（Plex、Emby）可能需要转码
- 移动设备支持不完整

### 2. 不是影视行业标准

**影视资源主流音频格式：**

| 格式 | 用途 | 质量 |
|------|------|------|
| **TrueHD / Atmos** | 高端蓝光 | 最高 |
| **DTS-HD MA** | 蓝光标准 | 无损 |
| **DTS / DTS-ES** | 蓝光、DVD | 高 |
| **AC3 (Dolby Digital)** | DVD 标准 | 中高 |
| **DDP (Dolby Digital Plus)** | 流媒体 | 中高 |
| **AAC** | 网络视频（WEB-DL） | 中 |
| **MP3** | 老格式 | 低 |
| **OPUS** | 网络通信 | 低（影视用途） |

**OPUS 主要应用场景：**
- YouTube 低码率视频
- Zoom / Teams 视频会议
- Discord 语音聊天
- WebRTC 实时通信
- 网络电台直播

### 3. PT 站规则
许多 PT 站点规定：
- 禁止上传 OPUS 编码的种子
- 或将 OPUS 标记为"低质量"
- 不符合收藏和长期保存需求

### 4. 制作质量指标
使用 OPUS 的资源通常意味着：
- 制作组追求小体积而非质量
- 可能是低质量压制
- 不适合高质量收藏

## 常见音频格式质量排序

```
高质量 ←——————————————————————————————————→ 低质量

TrueHD/Atmos > DTS-HD MA > DTS > AC3 > DDP > AAC > MP3 > OPUS
```

## 过滤规则配置

### 当前问题
原规则：
```json
"exclude": "(?i)OPUS"
```

**问题：** 会误匹配包含 "opus" 子串的任何词，如 "**Oct**o**pus**"（章鱼）

### 正确配置
使用单词边界 `\b`：
```json
"exclude": "(?i)\\bOPUS\\b"
```

**注意：** JSON 中反斜杠需要转义，写成 `\\b`（双反斜杠）

### 验证效果
- ✅ `2160p WEB-DL OPUS 5.1` - 会被过滤
- ✅ `1080p H265 OPUS` - 会被过滤
- ❌ `Octopus with Broken Arms` - 不会被过滤
- ❌ `The Octopus Teacher` - 不会被过滤

## 推荐配置

### 完整的 filterGlobal 规则
```json
{
  "id": "filterGlobal",
  "name": "filterGlobal",
  "include": "",
  "exclude": "(?i)日语无字|先行|\\bDV\\b|MiniBD|DIY原盘|iPad|UPSCALE|AV1|BDMV|RMVB|\\bDVD\\b|vcd|480p|\\bOPUS\\b",
  "seeders": ""
}
```

**说明：**
- 移除了 `HQ`（如果想要高质量版本）
- `OPUS` 改为 `\\bOPUS\\b`（避免误匹配）
- `DV` 改为 `\\bDV\\b`（避免误匹配）
- `DVD` 改为 `\\bDVD\\b`（更精确）

## 其他建议

### 如果想要杜比视界
杜比视界的缩写也是 DV（Dolby Vision），如果你想要杜比视界版本：
- 完全移除 `DV` 规则
- 或者创建专门的 DV 规则优先选择

### 如果想要高质量版本
- 移除 `HQ` 规则
- 或者创建 HQ 规则组优先选择高码率版本

## 相关资源

- [Opus 官方网站](https://opus-codec.org/)
- [RFC 6716 - Opus Audio Codec](https://tools.ietf.org/html/rfc6716)
- [维基百科 - Opus (音频格式)](https://zh.wikipedia.org/wiki/Opus_(%E9%9F%B3%E9%A2%91%E6%A0%BC%E5%BC%8F))

## 更新记录

- **2026-01-09** - 初始创建，记录 OPUS 相关信息和过滤规则配置
