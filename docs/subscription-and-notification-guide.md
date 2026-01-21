# MoviePilot 订阅与通知配置指南

本文档总结了 MoviePilot 使用过程中的关键配置要点和常见问题解决方案。

## 目录
- [目录配置与自动匹配](#目录配置与自动匹配)
- [通知系统配置](#通知系统配置)
- [整理流程详解](#整理流程详解)
- [下载器Webhook配置](#下载器webhook配置)
- [手动添加种子的处理](#手动添加种子的处理)
- [搜索关键词策略](#搜索关键词策略)
- [自定义识别词](#自定义识别词)
- [常见问题](#常见问题)

---

## 目录配置与自动匹配

### 核心逻辑

MoviePilot 在订阅时如果**不指定目录**，会根据以下规则自动匹配：

```python
# 匹配优先级（从高到低）
1. 媒体类型匹配（电影/电视剧）
2. 媒体类别匹配（国产剧/欧美剧/综艺等）
3. 存储类型匹配（本地/网络存储）
4. 同盘优先（源文件和目标在同一磁盘）
```

**重要**：如果有多个目录符合条件，会返回**配置顺序中的第一个**！

### 媒体类别选项

#### 电影类别
- 全部
- 动画电影
- 华语电影
- 外语电影

#### 电视剧类别
- 全部
- 国产剧
- 欧美剧
- 日韩剧
- 国漫
- 日番
- 综艺
- 纪录片
- 儿童
- 未分类

### 目录配置方案

#### 方案1：完整分类（精细化管理）

```
电影目录：
├── 动画电影（类别：动画电影）
├── 华语电影（类别：华语电影）
├── 外语电影（类别：外语电影）
└── 电影（类别：全部）← 兜底目录

电视剧目录：
├── 国产剧（类别：国产剧）
├── 欧美剧（类别：欧美剧）
├── 日韩剧（类别：日韩剧）
├── 国漫（类别：国漫）
├── 日番（类别：日番）
├── 综艺（类别：综艺）
├── 纪录片（类别：纪录片）
├── 儿童（类别：儿童）
└── TV（类别：全部）← 兜底目录
```

#### 方案2：简化分类（推荐）

```
电影目录：
└── 电影（类别：全部）

电视剧目录：
├── 国产剧（类别：国产剧）
├── 欧美剧（类别：欧美剧）
├── 日番（类别：日番）
├── 综艺（类别：综艺）
└── TV（类别：全部）← 兜底目录
```

### ⚠️ 关键注意事项

#### 1. 目录顺序至关重要

**正确顺序**：具体类别在前，"全部"在后
```
✅ 正确：
1. 国产剧（类别：国产剧）
2. 欧美剧（类别：欧美剧）
3. TV（类别：全部）← 放最后
```

**错误顺序**：会导致所有内容都匹配第一个目录
```
❌ 错误：
1. TV（类别：全部）← 放第一会拦截所有匹配
2. 国产剧（类别：国产剧）← 永远匹配不到
3. 欧美剧（类别：欧美剧）← 永远匹配不到
```

#### 2. 兜底目录必不可少

- 必须有一个"类别：全部"的目录作为兜底
- 当无法匹配具体类别时使用
- 放在同类型目录的最后位置

---

## 通知系统配置

### 通知层级关系

MoviePilot 的通知系统是**两层结构**：

```
第一层：全局通知开关（Telegram/微信等）
├── ☑ 资源下载
├── ☑ 整理入库  ← 全局开关
├── ☑ 订阅
└── ☑ 媒体服务器

第二层：目录级通知开关
├── 电影目录
│   └── ☑ 通知  ← 目录开关
└── TV目录
    └── ☐ 通知  ← 如果未勾选，此目录不发通知
```

### 通知发送条件

代码逻辑（transfer.py:410-422）：
```python
if transferinfo.need_notify and (task.background or not task.manual):
    # 发送通知
```

其中 `need_notify` 的值来源于：
```python
need_notify = target_directory.notify  # 目录配置中的通知开关
```

### 常见通知类型

| 通知类型 | 触发时机 | 通知开关 |
|---------|---------|---------|
| 资源下载 | 添加到下载器时 | 全局"资源下载"开关 |
| 订阅完成 | 订阅搜索完成时 | 全局"订阅"开关 |
| 整理入库 | 文件整理到媒体库时 | 全局"整理入库"**且**目录"通知"都启用 |
| 媒体服务器 | 媒体库刷新时 | 全局"媒体服务器"开关 |

### ⚠️ 常见问题：电影有通知，电视剧没有

**原因**：目录级通知开关未启用

**解决方法**：
1. 进入"设定" → "目录"
2. 展开电视剧相关目录
3. 勾选"☑ 通知"选项
4. 保存配置

---

## 整理流程详解

### 整理触发机制

MoviePilot 有两种方式触发下载文件的整理：

#### 1. 定时任务（兜底机制）

**代码位置**：`app/scheduler.py:311-321`

```python
# 下载器文件转移（每5分钟）
self._scheduler.add_job(
    self.start,
    "interval",
    id="transfer",
    name="下载文件整理",
    minutes=5,
    kwargs={'job_id': 'transfer'}
)
```

**执行流程**：
```
定时任务（每5分钟）
    ↓
调用 TransferChain().process()
    ↓
扫描所有下载器中已完成的种子
    ↓
检查种子是否在"下载器监控"目录中
    ↓
识别媒体信息
    ↓
整理到媒体库
    ↓
发送通知
```

**特点**：
- ✅ 可靠的兜底机制
- ⚠️ 最多延迟5分钟
- ✅ 不依赖外部触发

#### 2. 实时触发（Webhook机制）

通过配置下载器的webhook，在种子完成时立即触发整理。

**执行流程**：
```
种子下载完成
    ↓
下载器触发webhook
    ↓
调用 MoviePilot API: /api/v1/transfer/now
    ↓
立即执行 TransferChain().process()
    ↓
整理文件（同定时任务流程）
```

**特点**：
- ✅ 实时响应（几秒内）
- ✅ 用户体验更好
- ⚠️ 依赖下载器支持webhook

### 目录配置中的"自动整理"选项

**关键配置**：目录配置中的"自动整理"选项决定了整理流程是否生效。

#### 三种监控模式

##### 1. 下载器监控（推荐）

**适用场景**：通过下载器（qBittorrent/Transmission）下载的文件

**工作原理**：
```
TransferChain().process()
    ↓
获取下载器监控目录列表
    ↓
扫描下载器中已完成的种子
    ↓
检查种子文件是否在"资源目录"中
    ↓
如果在 → 整理到"媒体库目录"
```

**关键代码**（transfer.py:814-848）：
```python
# 获取下载器监控目录
download_dirs = DirectoryHelper().get_download_dirs()

# 如果没有下载器监控的目录则不处理
if not any(dir_info.monitor_type == "downloader" and dir_info.storage == "local"
           for dir_info in download_dirs):
    return True

# 检查文件是否在下载器监控目录中
for dir_info in download_dirs:
    if dir_info.monitor_type != "downloader":
        continue
    if file_path.is_relative_to(Path(dir_info.download_path)):
        is_downloader_monitor = True  # 在监控目录中
        break

if not is_downloader_monitor:
    continue  # 不在监控目录中，跳过
```

**配置示例**：
```
目录别名：电影
媒体类型：电影
自动整理：下载器监控  ← 关键配置
资源存储：本地
资源目录：/mnt/storage/downloads/movie  ← 下载器保存位置
媒体库存储：本地
媒体库目录：/mnt/storage/video/  ← 整理后的位置
整理方式：硬链接
```

##### 2. 目录监控

**适用场景**：直接监控文件系统，适合非下载器来源的文件

**工作原理**：
- 直接监控"资源目录"的文件变化
- 发现新文件立即整理
- 不检查下载器状态

##### 3. 兼容模式

同时启用下载器监控和目录监控。

### 整理流程详细步骤

**代码位置**：`app/chain/transfer.py:804-898`

#### 步骤1：获取已完成任务

```python
torrents = self.list_torrents(status=TorrentStatus.TRANSFER)
```

- 从所有配置的下载器获取已完成的种子
- 不限定特定下载器或种子

#### 步骤2：检查是否在监控目录

```python
for dir_info in download_dirs:
    if dir_info.monitor_type != "downloader":
        continue
    if file_path.is_relative_to(Path(dir_info.download_path)):
        is_downloader_monitor = True
        break
```

- 遍历所有"下载器监控"目录
- 检查种子文件是否在这些目录中
- 不在监控目录中的文件会被跳过

#### 步骤3：识别媒体信息

**通过MoviePilot下载的种子**（transfer.py:849-861）：
```python
downloadhis = DownloadHistoryOper().get_by_hash(torrent.hash)
if downloadhis:
    # 使用下载记录中的TMDBID/豆瓣ID识别
    mediainfo = self.recognize_media(
        mtype=MediaType(downloadhis.type),
        tmdbid=downloadhis.tmdbid,
        doubanid=downloadhis.doubanid
    )
```

**手动添加的种子**（transfer.py:660-661）：
```python
else:
    # 没有下载记录，通过文件名识别
    mediainfo = MediaChain().recognize_by_meta(task.meta)
```

#### 步骤4：执行整理

```python
state, errmsg = self.do_transfer(
    fileitem=FileItem(...),
    mediainfo=mediainfo,
    downloader=torrent.downloader,
    download_hash=torrent.hash,
    background=False
)
```

#### 步骤5：发送通知

如果配置了通知，会发送整理完成的通知（前提是目录启用了"通知"选项）。

---

## 下载器Webhook配置

### qBittorrent配置

**位置**：设置 → 下载 → 运行外部程序

**配置示例**：
```
☑ torrent 完成时运行外部程序

curl "http://192.168.2.5:3000/api/v1/transfer/now?token=YOUR_API_TOKEN"
```

**支持的参数**（区分大小写）：
- `%N`：Torrent 名称
- `%L`：分类
- `%G`：标签（以逗号分隔）
- `%F`：内容路径（与多文件 torrent 的根目录相同）

**注意**：虽然qBittorrent支持传递这些参数，但MoviePilot的 `/transfer/now` 接口目前**不使用**这些参数。

### API接口详解

**端点**：`GET /api/v1/transfer/now`

**代码位置**：`app/api/endpoints/transfer.py:181-187`

```python
@router.get("/now", summary="立即执行下载器文件整理", response_model=schemas.Response)
def now(_: Annotated[str, Depends(verify_apitoken)]) -> Any:
    """
    立即执行下载器文件整理 API_TOKEN认证（?token=xxx）
    """
    TransferChain().process()
    return schemas.Response(success=True)
```

#### 接口参数

**必需参数**：
- `token`: API认证token（从MoviePilot设置中获取API_TOKEN）

**可选参数**：
- 无（接口不接受其他参数）

#### 如何获取API_TOKEN

1. 登录MoviePilot
2. 进入"设定" → "基础设置"
3. 找到"API密钥"（API_TOKEN）
4. 复制token并替换配置中的 `YOUR_API_TOKEN`

### 处理范围

#### ✅ 会处理的内容

1. **所有下载器**
   - qBittorrent
   - Transmission
   - 等所有已配置的下载器

2. **所有已完成的种子**
   - 状态为 `TRANSFER`（已完成，未整理）的种子
   - 不限制特定的种子hash
   - 不限制特定的下载器

3. **所有"下载器监控"目录**
   - 遍历所有目录配置
   - 只要 `monitor_type == "下载器监控"`
   - 检查种子文件是否在这些目录中

#### 调用流程示意

```
调用 /transfer/now
    ↓
获取所有下载器中"已完成"的种子
    ├─ qBittorrent: 种子A（电影目录）
    ├─ qBittorrent: 种子B（电视剧目录）
    └─ Transmission: 种子C（电影目录）
    ↓
逐个检查种子是否在"下载器监控"目录中
    ├─ 种子A在 /mnt/storage/downloads/movie ✅ → 整理
    ├─ 种子B在 /mnt/storage/downloads/tv ✅ → 整理
    └─ 种子C不在任何监控目录 ❌ → 跳过
    ↓
完成
```

### Webhook优势

#### ✅ 实时性

```
定时任务：下载完成 → 等待0-5分钟 → 整理
Webhook：下载完成 → 几秒内 → 整理
```

#### ✅ 双重保障

- Webhook作为主要触发方式（实时）
- 定时任务作为兜底机制（如果webhook失败）

#### ✅ 不会重复整理

即使多次触发，整理历史记录会防止重复整理同一文件。

### 配置示例

#### 完整qBittorrent配置

```bash
# torrent完成时运行外部程序
curl "http://192.168.2.5:3000/api/v1/transfer/now?token=0eb67c0148720"
```

**效果**：
1. 种子下载完成
2. qBittorrent立即调用MoviePilot API
3. MoviePilot扫描所有已完成的种子
4. 识别并整理文件
5. 发送通知

---

## 手动添加种子的处理

### 场景说明

用户直接在PT站搜索种子，手动添加到下载器（qBittorrent），而不是通过MoviePilot的订阅功能下载。

### ✅ 会进行整理

**结论**：**会自动整理**

#### 整理流程

```
1. 手动添加种子到qBittorrent
   ↓
2. qBittorrent下载到：/mnt/storage/downloads/movie/电影名称
   ↓
3. 下载完成
   ↓
4. 触发整理（两种方式之一）：
   - Webhook: 立即触发（如果配置了）
   - 定时任务: 最多5分钟后触发
   ↓
5. TransferChain().process() 执行：
   - 扫描下载器中所有已完成的种子
   - 检查种子是否在"下载器监控"目录中 ✅
   ↓
6. 识别媒体信息（transfer.py:660-661）：
   # 没有下载记录，通过文件名识别
   mediainfo = MediaChain().recognize_by_meta(task.meta)
   ↓
7. 整理到媒体库：
   /mnt/storage/video/电影名称 (年份)/文件
   ↓
8. 发送通知（如果启用）
```

#### 关键代码

**检查下载记录**（transfer.py:849-870）：
```python
# 查询下载记录
downloadhis = DownloadHistoryOper().get_by_hash(torrent.hash)

if downloadhis:
    # 通过MoviePilot下载的，使用TMDBID识别
    mediainfo = self.recognize_media(
        tmdbid=downloadhis.tmdbid,
        doubanid=downloadhis.doubanid
    )
else:
    # 手动添加的种子，通过文件名识别
    mediainfo = None  # 稍后在__handle_transfer中识别
```

**文件名识别**（transfer.py:660-661）：
```python
# 没有下载记录，按文件识别
mediainfo = MediaChain().recognize_by_meta(task.meta)
```

### ❌ 不会更新订阅状态

**结论**：**不会自动更新订阅**

#### 原因分析

1. **整理完成事件**（transfer.py:482-489）：
   ```python
   # 发送整理完成事件
   self.eventmanager.send_event(EventType.TransferComplete, {
       'fileitem': task.fileitem,
       'meta': task.meta,
       'mediainfo': task.mediainfo,
       'transferinfo': transferinfo,
       'downloader': task.downloader,
       'download_hash': task.download_hash,
   })
   ```

2. **订阅模块未监听此事件**：
   - 检查了 `app/chain/subscribe.py`
   - 只发现监听 `EventType.SiteDeleted` 事件
   - **没有**监听 `EventType.TransferComplete` 事件

3. **订阅更新只在下载时触发**：
   - `finish_subscribe_or_not()` - 判断是否完成订阅
   - `__update_subscribe_note()` - 更新已下载集数
   - `__update_lack_episodes()` - 更新缺失集数
   - 这些方法只在**通过订阅搜索下载**时才会被调用

#### 实际影响

假设订阅了某剧的1-100集：

**场景1：通过订阅自动下载**
```
订阅搜索 → 下载1-50集 → 整理
    ↓
更新订阅：
- note字段记录已下载：[1-50]
- lack_episode更新：51-100
    ↓
继续搜索51-100集
```

**场景2：手动添加到下载器**
```
手动添加 → 下载1-50集 → 整理 ✅
    ↓
订阅状态不变：
- note字段仍为空：[]
- lack_episode仍为：1-100
    ↓
继续搜索1-100集（包括已下载的）❌
```

#### 解决方案

如果手动下载了部分集数，建议：

1. **手动更新订阅**
   - 在MoviePilot界面编辑订阅
   - 修改"开始集数"或手动更新已下载集数

2. **删除已完成的订阅**
   - 如果所有集数都下载完了
   - 手动删除订阅，避免重复搜索

3. **保持现状**
   - 整理功能正常工作
   - 订阅继续搜索（会有重复，但不影响使用）
   - 整理历史会防止重复整理

### 总结对比

| 功能 | 订阅自动下载 | 手动添加到下载器 |
|------|-------------|-----------------|
| 整理文件 | ✅ 会整理 | ✅ 会整理 |
| 识别媒体 | ✅ 使用TMDBID | ✅ 通过文件名 |
| 更新订阅已下载集数 | ✅ 会更新 | ❌ 不会更新 |
| 更新订阅缺失集数 | ✅ 会更新 | ❌ 不会更新 |
| 自动完成订阅 | ✅ 会完成 | ❌ 不会完成 |
| 发送通知 | ✅ 会发送 | ✅ 会发送 |

---

## 搜索关键词策略

### 核心原则

**搜索关键词应根据 PT 站的实际资源标题格式选择**

### 搜索流程

```
MoviePilot → 发送搜索关键词 → PT站搜索引擎 → 返回种子列表
                ↓
           关键阶段：PT站能否找到资源
```

### 常见场景

#### 场景1：PT站使用英文标题

**问题**：
- TMDB 标题：`声鸣远扬2025`（中文）
- PT 站资源：`Sound Trek S01 2025`（英文）
- 使用中文搜索 → PT站搜索不到或结果很少 ❌

**解决方法**：
```
订阅时手动指定搜索关键词：Sound Trek
```

#### 场景2：标题包含年份

**问题**：
- 搜索：`声鸣远扬2025` → 结果少
- 搜索：`声鸣远扬` → 结果多

**解决方法**：
```
搜索关键词不要带年份（除非必要）
✅ 推荐：Sound Trek
❌ 避免：Sound Trek 2025
```

### 最佳实践

1. **查看 PT 站实际资源标题格式**
   - 如果主要是英文 → 用英文关键词
   - 如果主要是中文 → 用中文关键词

2. **手动测试不同关键词**
   - 在 MoviePilot 手动搜索页面测试
   - 比较搜索结果数量和准确度
   - 选择最佳的关键词

3. **订阅时指定搜索关键词**
   - 在订阅编辑界面的"搜索关键词"字段
   - 输入测试好的关键词

---

## 自定义识别词

### 使用场景

当 PT 站资源的季数标注与 TMDB 数据不一致时使用。

### 语法格式

```
基础替换：
被替换词 => 替换词

正则表达式：
使用 Python re 模块语法
. 匹配任意字符（需要转义：\.）
```

### 实际案例

#### 案例1：斗罗大陆II 绝世唐门

**问题**：
- PT 站资源：`Soul Land S02E131`（标记为第2季）
- TMDB 数据：第1季
- 订阅第1季 → 搜索到但匹配失败

**解决方法**：
```
自定义识别词：
Soul.Land.S02 => Soul.Land.S01

或更精确（推荐）：
Soul\.Land\.S02E => Soul.Land.S01E
```

#### 案例2：点号 vs 空格

**标准写法**（更精确）：
```
Soul\.Land\.S02 => Soul\.Land\.S01
说明：\. 只匹配字面点号
```

**简化写法**（也可以）：
```
Soul.Land.S02 => Soul.Land.S01
说明：. 匹配任意字符，但在这个场景下也能工作
```

### ⚠️ 重要提醒

**自定义识别词只在搜索阶段生效**

- ✅ 搜索时：应用识别词，能找到资源
- ❌ 整理时：可能不应用，导致按原始季数整理

**解决方法**：
- 在订阅设置中指定"自定义类别"或"目标路径"
- 或者修改订阅，直接订阅对应的季数

---

## 常见问题

### Q1: 订阅搜索不到资源

**可能原因**：
1. 搜索关键词不匹配 PT 站资源标题
2. 年份匹配失败（TMDB season_years 数据不完整）
3. 过滤规则过于严格

**排查步骤**：
1. 手动搜索测试不同关键词
2. 查看 PT 站实际资源标题格式
3. 指定搜索关键词（英文/中文/不带年份）
4. 检查日志中的匹配失败原因

### Q2: 搜索到资源但匹配失败

**可能原因**：
1. 年份不匹配
2. 季数不匹配
3. 标题识别失败

**解决方法**：
1. 使用自定义识别词修正季数
2. 指定搜索关键词
3. 检查 TMDB 数据是否正确

### Q3: 电影有通知，电视剧没有

**原因**：目录级通知开关未启用

**解决方法**：
在"设定" → "目录"中，展开电视剧目录，勾选"通知"选项

### Q4: 整理后文件在错误的目录

**可能原因**：
1. 目录配置顺序错误（"全部"在前）
2. 没有匹配的具体类别目录
3. 自定义识别词在整理阶段未生效

**解决方法**：
1. 调整目录配置顺序（具体类别在前，"全部"在后）
2. 添加对应的类别目录
3. 在订阅中指定目标目录或类别

### Q5: 订阅搜索到资源但都被过滤（大小不匹配）

**典型日志**：
```
【DEBUG】种子 憨憨 - Kingdom S01 2160p ... 大小 12.27G 不在范围 0-10000MB
【WARNING】尸战朝鲜 没有符合过滤规则的资源
```

**原因分析**：

过滤规则中的 `size_range` 设计原理：

1. **理论上**：`filterSeries` 的 `size_range` 是限制**单集大小**（MB）

   **代码逻辑**（`app/modules/filter/__init__.py:417-427`）：
   ```python
   def __match_size(torrent: TorrentInfo, size_range: str) -> bool:
       """
       判断种子是否匹配大小范围（MB），剧集拆分为每集大小
       """
       # 从标题中解析集数
       meta = MetaInfo(title=torrent.title, subtitle=torrent.description)
       episode_count = meta.total_episode or 1  # 默认为1
       # 计算单集大小
       torrent_size = torrent.size / episode_count
   ```

2. **实际上**：对于**整季包**（标题中没有集数信息），会被当作 1 集处理

   **示例**：
   ```
   种子标题：Kingdom S01 2160p Netflix WEB-DL DDP5.1 H265-HHWEB
   解析结果：只有季信息（S01），没有集数范围（如 E01-E06）
   episode_count = 1  ← 默认值
   单集大小 = 12.27GB ÷ 1 = 12.27GB
   12.27GB > 10GB 限制 → 被过滤 ❌
   ```

3. **为什么没有集数信息**：
   - 整季包通常只标注 `S01`、`S02`，不标注具体集数
   - 系统无法从标题解析出总集数
   - 代码默认 `episode_count = 1`

**解决方法**：

**方案1：调整 filterSeries 大小限制**（推荐）

修改 `docs/rule.json` 中的 `filterSeries` 规则：

```json
{
  "id": "filterSeries",
  "name": "filterSeries",
  "include": "",
  "exclude": "",
  "size_range": "0-20000",  // 改为 20GB，适配整季包
  "seeders": ""
}
```

**建议值**：
- 1080p 整季包：`0-20000`（20GB）
- 4K 整季包：`0-30000`（30GB）或更大
- 单集资源：`0-10000`（10GB）

**方案2：创建单独的规则**

如果需要区分整季包和单集：

```json
// 单集规则（严格限制）
{
  "id": "filterSingle",
  "size_range": "0-7000"
}

// 剧集规则（宽松限制，适配整季包）
{
  "id": "filterSeries",
  "size_range": "0-30000"
}
```

**注意事项**：

1. **单位是 MB**：`10000` = 10GB，`20000` = 20GB
2. **影响所有剧集**：调整后会影响所有使用 `filterSeries` 的规则组
3. **建议测试**：先用较大的值（如 `30000`），观察一段时间后再调整

---

## 最佳实践总结

### 1. 目录配置
- ✅ 具体类别目录在前，"全部"在后
- ✅ 必须有兜底目录（类别：全部）
- ✅ 每个目录都启用"通知"选项
- ✅ 根据实际需求简化分类，不必过度细分

### 2. 订阅配置
- ✅ 根据 PT 站资源格式选择搜索关键词
- ✅ 不要带年份（除非必要）
- ✅ 测试后再添加订阅
- ✅ 复杂情况使用自定义识别词

### 3. 通知配置
- ✅ 全局通知开关：启用"整理入库"
- ✅ 目录通知开关：每个目录都启用"通知"
- ✅ 测试验证：手动下载一个文件测试

### 4. 问题排查
- ✅ 查看日志定位问题
- ✅ 手动搜索测试关键词
- ✅ 检查 PT 站资源实际格式
- ✅ 逐步排除可能的原因

---

## 参考代码

### 目录匹配逻辑
```python
# app/helper/directory.py:54-112
def get_dir(self, media: MediaInfo, ...):
    # 1. 根据媒体类型和类别匹配
    # 2. 如果有多个匹配，返回第一个
    # 3. 优先同盘目录
    return matched_dirs[0]
```

### 通知发送条件
```python
# app/chain/transfer.py:410-422
if transferinfo.need_notify and (task.background or not task.manual):
    self.send_transfer_message(...)

# app/modules/filemanager/__init__.py:446
need_notify = target_directory.notify
```

### 自定义识别词处理
```python
# app/core/meta/words.py:16-68
def prepare(self, title: str, custom_words: List[str] = None):
    # 应用自定义识别词
    # 支持替换词、屏蔽词、集偏移
    return title, applied_words
```

---

## 更新日志

- 2026-01-11: 新增"Q5: 订阅搜索到资源但都被过滤（大小不匹配）"常见问题，详细说明 filterSeries 大小限制的计算逻辑
- 2025-12-20: 新增"整理流程详解"、"下载器Webhook配置"、"手动添加种子的处理"章节
- 2025-12-15: 初始版本，总结订阅、通知、目录配置等核心知识点

---

## 相关文档

- [MoviePilot 官方文档](https://github.com/jxxghp/MoviePilot)
- [开发环境搭建](./development-setup.md)
- [MCP API 文档](./mcp-api.md)
