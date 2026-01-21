# MoviePilot 订阅流程详细分析

> 本文档提供了 MoviePilot 订阅系统的完整技术分析，包括数据模型、调度机制、搜索流程、过滤规则等核心功能的详细说明。

## 目录

- [1. 订阅数据模型](#1-订阅数据模型)
- [2. 订阅生命周期管理](#2-订阅生命周期管理)
- [3. 订阅调度系统](#3-订阅调度系统)
- [4. 订阅搜索资源完整流程](#4-订阅搜索资源完整流程)
- [5. 过滤规则系统详解](#5-过滤规则系统详解)
- [6. 季集匹配和缺失检测](#6-季集匹配和缺失检测)
- [7. 已下载集数处理](#7-已下载集数处理)
- [8. 完成订阅逻辑](#8-完成订阅逻辑)
- [9. 订阅信息同步](#9-订阅信息同步)
- [10. 关键文件映射](#10-关键文件映射)
- [11. 订阅流程时序图](#11-订阅流程时序图)
- [12. 高级特性说明](#12-高级特性说明)
- [13. 性能优化机制](#13-性能优化机制)
- [14. 重要限制说明](#14-重要限制说明)

---

## 1. 订阅数据模型

### 数据库模型

**位置**: `app/db/models/subscribe.py`

### Subscribe 表结构

#### 基础字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `id` | Integer | 主键 |
| `name` | String | 订阅标题 |
| `year` | String | 年份 |
| `type` | String | 媒体类型（TV/MOVIE） |
| `keyword` | String | 搜索关键字 |
| `username` | String | 订阅用户 |

#### 媒体ID字段

| 字段 | 类型 | 说明 |
|------|------|------|
| `tmdbid` | Integer | TMDB ID |
| `imdbid` | String | IMDB ID |
| `tvdbid` | Integer | TVDB ID |
| `doubanid` | String | 豆瓣ID |
| `bangumiid` | Integer | Bangumi ID |
| `mediaid` | String | 媒体ID（未知来源） |

#### 季集信息（仅电视剧）

| 字段 | 类型 | 说明 |
|------|------|------|
| `season` | Integer | 季号 |
| `total_episode` | Integer | 总集数 |
| `start_episode` | Integer | 开始集数 |
| `lack_episode` | Integer | 缺失集数 |
| `episode_group` | String | 剧集组选择 |

#### 下载配置

| 字段 | 类型 | 说明 |
|------|------|------|
| `include` | String | 包含关键字 |
| `exclude` | String | 排除关键字 |
| `quality` | String | 质量过滤规则 |
| `resolution` | String | 分辨率 |
| `effect` | String | 特效 |
| `filter` | String | 过滤规则（已废弃） |
| `filter_groups` | String | 过滤规则组（JSON数组） |
| `best_version` | Integer | 是否洗版（1=是） |
| `current_priority` | Integer | 当前优先级（用于洗版） |

#### 保存路径和下载器

| 字段 | 类型 | 说明 |
|------|------|------|
| `save_path` | String | 保存路径 |
| `sites` | String | 订阅站点（JSON数组） |
| `downloader` | String | 下载器名称 |

#### 状态和时间

| 字段 | 类型 | 说明 |
|------|------|------|
| `state` | String | 订阅状态（N/R/P/S） |
| `last_update` | String | 最后更新时间 |
| `date` | String | 创建时间 |

**订阅状态说明**:
- `N`: 新建（未处理）
- `R`: 订阅中（正在搜索）
- `P`: 待定（信息待更新，允许搜索，不允许完成）
- `S`: 暂停（不参与任何动作）

#### 附加信息

| 字段 | 类型 | 说明 |
|------|------|------|
| `note` | String | 已下载集数（JSON） |
| `custom_words` | String | 自定义识别词 |
| `media_category` | String | 自定义媒体类别 |
| `search_imdbid` | Integer | 是否使用IMDBID搜索 |
| `manual_total_episode` | Integer | 是否手动修改过总集数 |
| `poster` | String | 海报图片 |
| `backdrop` | String | 背景图片 |
| `description` | String | 描述 |
| `vote` | Float | 评分 |

### Schema模型

**位置**: `app/schemas/subscribe.py`

---

## 2. 订阅生命周期管理

### 操作类

**位置**: `app/db/subscribe_oper.py`

### 2.1 订阅创建流程

**关键文件**: `app/chain/subscribe.py`
**关键方法**: `SubscribeChain.add()`, `SubscribeChain.async_add()`

```python
流程步骤:
1. 媒体识别
   - 根据 TMDBID/豆瓣ID/MediaID 识别媒体
   - 如果都没有，使用标题识别（名称兜底）

2. 总集数计算（仅电视剧）
   - 获取媒体的季集信息
   - 如果未指定总集数，从 TMDB 获取
   - 初始化缺失集数 = 总集数

3. 获取默认配置
   - 从系统配置获取默认质量、分辨率、过滤规则等
   - 可被用户指定的参数覆盖

4. 创建订阅
   - 检查是否已存在（按 TMDBID/豆瓣ID + 季号）
   - 插入数据库，状态为 'N'（新建）
```

### 2.2 订阅状态转换

```
N (新建)
├─→ 首次搜索触发 → R (订阅中) → 搜索完成/媒体已存在 → 删除 + 历史记录

R (订阅中)
├─→ 定期搜索
├─→ 匹配到资源并下载 → 缺失集数更新
└─→ 所有集都下载/媒体存在 → 删除 + 历史记录

P (待定)
├─→ 信息更新中（允许搜索）
└─→ 信息更新完成 → R

S (暂停)
└─→ 手动取消暂停 → R
```

---

## 3. 订阅调度系统

### 调度器

**位置**: `app/scheduler.py`

### 3.1 定时任务配置

| 任务ID | 任务名称 | 触发频率 | 说明 |
|--------|---------|---------|------|
| `subscribe_search` | 订阅搜索补全 | 可配置间隔(默认24h) | 已订阅内容搜索 |
| `new_subscribe_search` | 新增订阅搜索 | 每5分钟 | N状态订阅的首次搜索 |
| `subscribe_tmdb` | 订阅元数据更新 | 每6小时 | 更新TMDB媒体信息 |
| `subscribe_refresh` | 订阅刷新 | RSS模式：每30分钟<br>Spider模式：32次随机时间 | 刷新站点资源 |
| `subscribe_follow` | 关注的订阅分享 | 每1小时 | 自动添加关注用户的分享 |

### 3.2 调度执行机制

```python
# 调度器启动方式:
1. BackgroundScheduler (APScheduler)
   - 线程池执行 (ThreadPoolExecutor)
   - 支持协程函数和普通函数

2. 执行方法 (Scheduler.start):
   - 检查任务是否已运行（防止并发）
   - 协程函数使用 asyncio.run_coroutine_threadsafe
   - 支持多进程模式

3. 任务锁机制:
   - _rlock: 重入锁 (3600*2 秒超时)
   - 防止多个搜索任务同时进行
```

---

## 4. 订阅搜索资源完整流程

### 搜索链

**位置**: `app/chain/search.py`

### 4.1 搜索过程 (SubscribeChain.search)

```
步骤1: 获取待搜索订阅
├─ 按状态获取 (N/R/P)
└─ 检查创建时间(< 1分钟则跳过，留编辑时间)

步骤2: 媒体信息补充
├─ 生成元数据 (MetaInfo)
└─ 识别媒体详情 (MediaInfo)

步骤3: 检查媒体是否已存在
├─ 调用 DownloadChain.get_no_exists_info()
│  └─ 查询媒体库，返回缺失季集信息
├─ 判断是否已下载完毕
│  └─ 洗版模式: 检查优先级是否达到100
│  └─ 普通模式: 检查是否有缺失集
└─ 如果已完成 → 完成订阅 (删除+历史记录)

步骤4: 获取搜索参数
├─ 站点范围 (get_sub_sites)
├─ 过滤规则组 (SystemConfig/Subscribe级别)
├─ 自定义识别词 (custom_words)
└─ 过滤参数 (quality/resolution/effect等)

步骤5: 搜索资源 (SearchChain.process)
├─ 多关键字搜索 (title/original_title/别名等)
│  ├─ 1-10秒随机休眠（避免请求过快）
│  └─ 有结果即停止（除非配置搜索多名称）
├─ 季集过滤 (match_season_episodes)
│  └─ 只保留缺失季集的资源
└─ 规则过滤 (filter_torrents)
   └─ 应用过滤规则组

步骤6: 资源匹配与下载 (DownloadChain.batch_download)
├─ 电影: 直接下载
└─ 电视剧: 整季匹配 + 集数选择
   ├─ 查找满足整季的资源
   └─ 按优先级下载

步骤7: 更新订阅状态
├─ 记录已下载集数 (note字段)
├─ 更新缺失集数 (lack_episode)
├─ 更新最后更新时间
└─ 检查是否应完成订阅
```

### 4.2 资源刷新流程 (SubscribeChain.refresh/match)

**触发**: RSS/Spider模式定时刷新

```
流程:
1. 获取所有活跃订阅的订阅站点
   └─ 避免刷新无订阅站点的资源

2. 刷新站点最新资源 (TorrentsChain.refresh)
   └─ 缓存资源供后续匹配

3. 批量预识别种子 (match方法)
   ├─ 尝试识别未识别的种子
   ├─ 最多3次失败后放弃识别
   └─ 清理缓存

4. 订阅匹配 (对每个订阅):
   ├─ 生成媒体元数据
   ├─ 检查媒体是否已存在
   └─ 遍历缓存资源进行匹配

5. 资源匹配规则:
   ├─ 站点范围过滤
   ├─ 自定义识别词重新识别
   ├─ TMDBID/豆瓣ID直接比对
   ├─ 标题智能匹配 (match_torrent)
   └─ 季集匹配 + 规则过滤

6. 批量下载 (batch_download)
```

---

## 5. 过滤规则系统详解

### 过滤模块

**位置**: `app/modules/filter/__init__.py`

### 5.1 规则组结构

```python
# 规则组格式: 多个规则用 > 分隔，表示优先级从高到低

例: "SPECSUB & CNVOI & 4K & !BLU > CNSUB & CNVOI & 4K & !BLU > 4K & !BLU"

优先级:
- 第一个规则匹配 → 优先级 100
- 第二个规则匹配 → 优先级 99
- 以此类推...
- 都不匹配 → 被过滤掉
```

### 5.2 规则语法

| 符号 | 含义 | 例子 |
|------|------|------|
| `&` | AND (与) | `4K & HDR` |
| `\|` | OR (或) | `4K \| 1080P` |
| `!` | NOT (非) | `!BLU` (不包含蓝光) |

### 5.3 内置规则集

| 规则名 | 说明 | 匹配方式 |
|--------|------|---------|
| `BLU` | 蓝光原盘 | 正则匹配标题/描述 |
| `4K` | 4K分辨率 | `4k\|2160p\|x2160` |
| `1080P` | 1080P分辨率 | `1080[pi]\|x1080` |
| `720P` | 720P分辨率 | `720[pi]\|x720` |
| `CNSUB` | 中字 | 字幕识别 |
| `CNVOI` | 国语配音 | 语言识别 |
| `DOLBY` | 杜比视界 | `Dolby Vision\|DOVI\|DV` |
| `ATMOS` | 杜比全景声 | `Dolby Atmos\|Atmos` |
| `HDR` | HDR | `HDR\|HDR10\|HDR10+` |
| `SDR` | SDR | `SDR` |
| `REMUX` | REMUX | 编码格式 |
| `WEBDL` | WEB-DL | `WEB-?DL\|WEB-?RIP` |
| `H264` | H264编码 | `[Hx].?264\|AVC` |
| `H265` | H265编码 | `[Hx].?265\|HEVC` |
| `FREE` | 免费下载 | 下载系数为0 |
| `60FPS` | 高帧率 | `60fps\|60帧` |
| `3D` | 3D | `3D` |

### 5.4 规则匹配过程

**位置**: `app/modules/filter/__init__.py:282-376`

```python
def __match_rule(torrent: TorrentInfo, rule_name: str) -> bool:
    """
    规则匹配步骤:
    """
    规则配置 = rule_set[rule_name]

    # 1. TMDB规则检查 (可选)
    if 媒体信息符合TMDB规则:
        return True  # 直接匹配成功

    # 2. 匹配内容准备
    content = f"{torrent.title} {torrent.description} {labels}"
    if match_fields:  # 如果指定了匹配字段
        content = 只提取指定字段的内容

    # 3. 包含规则 (include)
    if includes 存在 且 content中不包含任何includes:
        return False

    # 4. 排除规则 (exclude)
    for exclude in excludes:
        if content中包含exclude:
            return False

    # 5. 大小范围检查
    if size_range:
        if torrent.size 不在size_range内:
            return False

    # 6. 做种人数检查
    if min_seeders:
        if torrent.seeders < min_seeders:
            return False

    # 7. FREE规则检查
    if downloadvolumefactor is not None:
        if torrent.downloadvolumefactor != downloadvolumefactor:
            return False

    # 8. 发布时间检查
    if publish_time:
        if torrent发布时间不在指定范围:
            return False

    return True  # 全部通过
```

### 5.5 附加参数过滤

**位置**: `app/chain/subscribe.py` - `get_params()`

```python
# 订阅级别的过滤参数

包含参数:
- include: 包含关键字
- exclude: 排除关键字
- quality: 质量级别
- resolution: 分辨率
- effect: 特效
- tv_size: 电视剧大小限制
- movie_size: 电影大小限制
- min_seeders: 最小做种人数
- min_seeders_time: 最小做种时间

获取优先级:
1. 订阅自定义参数
2. 系统配置默认参数 (SubscribeDefaultParams)
```

---

## 6. 季集匹配和缺失检测

### 相关代码

- `app/chain/download.py:428-541` - `get_no_exists_info()`
- `app/helper/torrent.py:154-229` - `match_season_episodes()`

### 6.1 缺失信息查询

**方法**: `DownloadChain.get_no_exists_info()`

```python
# 流程:
1. 媒体类型判断
   ├─ 电影: 查询是否存在，存在返回True/{}，否则返回False/{}
   └─ 电视剧: 按季按集查询缺失

2. 电视剧缺失检测 (针对每季):
   ├─ 获取TMDB季集列表
   ├─ 查询媒体库已存在的集
   ├─ 计算缺失集 (差集)
   └─ 组织成 NotExistMediaInfo 结构

3. NotExistMediaInfo 数据结构:
   {
       season: int,                    # 季号
       episodes: [1,2,3],             # 缺失集数列表 (空=整季缺失)
       total_episode: 12,              # 该季总集数
       start_episode: 1                # 开始集号
   }

4. 返回格式:
   {
       tmdbid_or_doubanid: {
           season_number: NotExistMediaInfo,
           ...
       },
       ...
   }
```

### 6.2 季集匹配

**方法**: `TorrentHelper.match_season_episodes()`

```python
# 检查种子资源是否包含缺失的季集

检查步骤:
1. 提取种子的季号列表 (meta.season_list)
   └─ 无季号则默认为 [1]

2. 提取种子的集号列表 (meta.episode_list)

3. 对于指定季号的订阅:
   ├─ 如果种子有集数且集数与缺失集数无交集 → 排除
   └─ 如果种子无集数 (整季) → 接受

4. 返回是否应该下载该资源
```

### 6.3 洗版特殊处理

```python
# best_version = 1 时的特殊逻辑:

1. 资源过滤:
   └─ 非整季资源被排除

2. 优先级限制:
   └─ 新资源优先级必须 > 已下载优先级

3. 完成条件:
   └─ current_priority == 100 (达到最高优先级)

4. 更新机制:
   ├─ 每次成功下载更新 current_priority
   └─ 洗版完成后删除订阅
```

---

## 7. 已下载集数处理

**位置**: `app/chain/subscribe.py`

### 7.1 记录已下载

**方法**: `SubscribeChain.__update_subscribe_note()`

```python
流程:
1. 获取已下载上下文列表
2. 提取元数据 (meta.episode_list)
   ├─ 电视剧: 直接使用集号列表
   └─ 电影: 使用 [1]
3. 合并到订阅的 note 字段 (去重)
4. 更新数据库

note字段示例: [1, 2, 3, 5, 7]  # 表示这些集数已下载
```

### 7.2 获取已下载

**方法**: `SubscribeChain.__get_downloaded()`

```python
返回值:
- 洗版状态: [] (始终返回空，忽略已下载)
- 普通模式: note字段内容 (已下载集号列表)
```

### 7.3 缺失集数更新逻辑

**方法**: `SubscribeChain.__get_subscribe_no_exits()`

```python
场景1: 订阅指定了总集数/开始集数
├─ 重新计算缺失集数范围
└─ 示例: 总12集，开始第2集 → 范围为[2,13]

场景2: 订阅有已下载集数
├─ 从缺失集中去掉已下载的
└─ 示例: 缺失[1-12]，已下载[2,4,6] → 剩余[1,3,5,7-12]

优先级:
1. 应用开始/总集数范围
2. 应用已下载排除
3. 如果交集为空，标记订阅已完成
```

---

## 8. 完成订阅逻辑

### 关键方法

- `SubscribeChain.finish_subscribe_or_not()` - `app/chain/subscribe.py:1127-1196`
- `SubscribeChain.__finish_subscribe()` - `app/chain/subscribe.py:1198-1235`

### 8.1 完成条件判断

```python
def finish_subscribe_or_not(subscribe, meta, mediainfo, downloads, lefts, force):
    """
    判断是否应该完成订阅
    """

    no_lefts = 是否无剩余缺失集

    if not subscribe.best_version:  # 普通模式
        # 更新已下载信息
        更新 note 字段

        # 更新缺失集数
        更新 lack_episode

        # 完成条件:
        if ((no_lefts and 电视剧) or (downloads and 电影) or force):
            完成订阅()
        else:
            继续订阅()

    elif downloads:  # 洗版模式且有下载
        更新优先级()

    elif current_priority == 100:  # 洗版完成
        完成订阅()

    else:  # 洗版继续
        继续搜索()
```

### 8.2 订阅完成操作

```python
def __finish_subscribe(subscribe, meta, mediainfo):
    """
    完成订阅的操作:
    """
    1. 新增订阅历史记录 (subscribe_history)
    2. 删除原订阅记录
    3. 发送完成通知消息
    4. 触发事件 (EventType.SubscribeComplete)
    5. 统计数据上报
```

---

## 9. 订阅信息同步

### 9.1 元数据更新

**方法**: `SubscribeChain.check()`
**触发**: 每6小时执行一次

```python
步骤:
1. 遍历所有订阅
2. 重新识别媒体信息
3. 更新字段:
   ├─ name: 标题
   ├─ year: 年份
   ├─ vote: 评分
   ├─ poster/backdrop: 图片
   ├─ description: 描述
   ├─ imdbid/tvdbid: IDs
   ├─ total_episode: 总集数 (如未手动修改)
   └─ lack_episode: 缺失集数 (根据总集数变化调整)
```

### 9.2 关注分享同步

**方法**: `SubscribeChain.follow()`
**触发**: 每1小时执行一次

```python
流程:
1. 获取关注用户列表 (SystemConfig)
2. 获取分享的订阅列表
3. 筛选:
   ├─ 来自关注用户的分享
   ├─ 本地未存在的订阅
   └─ 历史未订阅过的
4. 自动添加新订阅
```

---

## 10. 关键文件映射

| 功能 | 主要文件 | 关键方法 |
|------|---------|---------|
| **订阅管理** | `app/chain/subscribe.py` | `add()`, `search()`, `match()`, `refresh()` |
| **订阅数据库** | `app/db/subscribe_oper.py` | `add()`, `get()`, `list()`, `update()` |
| **订阅模型** | `app/db/models/subscribe.py` | Subscribe, SubscribeHistory |
| **搜索处理** | `app/chain/search.py` | `process()`, `__parse_result()` |
| **下载处理** | `app/chain/download.py` | `batch_download()`, `get_no_exists_info()` |
| **过滤模块** | `app/modules/filter/__init__.py` | `filter_torrents()`, `__match_rule()` |
| **种子匹配** | `app/helper/torrent.py` | `match_torrent()`, `match_season_episodes()` |
| **调度器** | `app/scheduler.py` | `init()`, `start()` |
| **API接口** | `app/api/endpoints/subscribe.py` | 各种订阅API端点 |

---

## 11. 订阅流程时序图

```
用户添加订阅
    ↓
SubscribeChain.add()
    ├─ 媒体识别
    ├─ 获取TMDB信息
    ├─ 计算总集数
    ├─ 应用默认配置
    └─ 创建订阅 (state='N')
        ↓
定时任务触发 (每5分钟)
    ├─ new_subscribe_search
    │   └─ SubscribeChain.search(state='N')
    │       ├─ 检查创建时间
    │       ├─ 获取缺失信息
    │       ├─ 搜索资源
    │       ├─ 过滤规则
    │       └─ 批量下载
    │           └─ 更新 state='R'
    └─ subscribe_search (24小时)
        └─ SubscribeChain.search(state='R')
            └─ (同上)

            或 subscribe_refresh (定期)
        └─ SubscribeChain.refresh()
            └─ SubscribeChain.match()
                └─ (缓存匹配，流程同search)

下载资源
    ↓
更新缺失集数/优先级
    ↓
判断是否应完成
    ├─ 普通模式: 无缺失且电视剧 或 有下载且电影
    └─ 洗版模式: 优先级==100
        ↓
    完成订阅
    ├─ 新增历史记录
    ├─ 删除原订阅
    ├─ 发送完成消息
    └─ 触发完成事件
```

---

## 12. 高级特性说明

### 12.1 自定义识别词 (custom_words)

```python
# 订阅级别的自定义识别词
# 用于重新识别种子元数据

当订阅设置了 custom_words 时:
1. 搜索阶段使用这些识别词
2. 缓存匹配时也会应用
3. 可用于处理特殊名称的媒体
```

### 12.2 剧集组选择 (episode_group)

```python
# 某些媒体有多个集数版本 (如绝版/特别版)

用途:
- 从不同的集数组中选择
- 与 TMDB episode_group 关联
- 影响总集数和缺失集计算
```

### 12.3 媒体类别自定义 (media_category)

```python
# 覆盖媒体库识别的类别
# 用于某些特殊分类的媒体
```

### 12.4 IMDBID搜索 (search_imdbid)

```python
# 某些站点支持使用 IMDBID 进行精确搜索
# 比标题搜索更准确但覆盖面小
```

---

## 13. 性能优化机制

### 13.1 搜索优化

- **多关键字搜索**: 使用 TMDB 标题、原标题、别名等多个关键字搜索
- **搜索停止**: 有结果时即停止搜索 (可配置)
- **随机休眠**: 1-10秒随机休眠，避免请求过快

### 13.2 缓存机制

- **种子缓存**: `TorrentsChain.refresh()` 缓存资源供后续匹配
- **预识别**: `match()` 预先识别未识别的种子
- **最多3次失败**: 识别失败3次后放弃

### 13.3 并发控制

- **锁机制**: 使用 RLock 防止搜索和匹配并发
- **任务检查**: 前置检查防止重复执行
- **停止检查**: 定期检查系统停止状态

---

## 14. 常见问题排查

### 14.1 订阅无法匹配资源

**可能原因**:

1. **季数不匹配**
   - 订阅的是第1季，但资源标注为第2季
   - 检查 TMDB 上的季度划分是否与资源一致

2. **分辨率不匹配**
   - 附加参数中设置了 `4K|2160p`，但搜索到的都是 1080p 资源
   - 检查订阅的分辨率设置

3. **规则组过滤**
   - 资源不符合规则组的任何优先级规则
   - 检查规则组配置，例如要求 HHWEB 但资源是 ADWeb

4. **站点范围**
   - 资源所在站点不在订阅的站点列表中
   - 检查订阅的站点设置

### 14.2 订阅无法完成

**可能原因**:

1. **缺失集数计算错误**
   - 已下载集数未正确记录
   - 检查 `note` 字段中的已下载集数

2. **总集数设置错误**
   - 手动设置的总集数与实际不符
   - 重新识别或手动修正总集数

3. **洗版模式未达到最高优先级**
   - `current_priority` 未达到 100
   - 等待更高优先级的资源

### 14.3 订阅重复下载

**可能原因**:

1. **已下载集数未更新**
   - 下载后未正确记录到 `note` 字段
   - 检查下载链的更新逻辑

2. **媒体库扫描不及时**
   - 缺失检测未能发现已存在的集
   - 手动触发媒体库刷新

---

## 15. 总结

MoviePilot 的订阅系统是一个功能完整、设计精良的自动化下载系统。它通过以下机制实现了高效、准确的资源订阅和下载：

1. **灵活的数据模型**: 支持电影和电视剧的各种订阅需求
2. **强大的调度系统**: 多种定时任务协同工作
3. **智能的搜索机制**: 多关键字、多站点、缓存匹配
4. **精确的过滤规则**: 支持复杂的优先级规则组合
5. **准确的季集匹配**: 精确检测缺失和已下载的集数
6. **完善的状态管理**: 从创建到完成的完整生命周期

代码结构清晰，模块职责分明，扩展性强，是一个优秀的开源项目。

---

---

## 14. 重要限制说明

### 14.1 搜索只返回第一页结果

**关键发现**: MoviePilot 在订阅搜索时**只搜索第一页（page=0）**，没有自动翻页功能。

#### 代码位置

**文件**: `app/chain/search.py`

**调用位置**: Line 386-391

```python
# 订阅搜索时
results = self.__search_all_sites(
    mediainfo=mediainfo,
    keyword=search_word,
    sites=sites,
    area=area
    # ← 注意：没有传递 page 参数
) or []
```

**默认值**: Line 496

```python
def __search_all_sites(self, keyword: str,
                       mediainfo: Optional[MediaInfo] = None,
                       sites: List[int] = None,
                       page: Optional[int] = 0,  # ← 默认第一页
                       area: Optional[str] = "title"):
```

#### 搜索流程

```
订阅搜索 → SearchChain.process()
    ↓
调用 __search_all_sites(page=0)  ← 固定第一页
    ↓
发送请求到 PT 站: torrents.php?...&page=0
    ↓
获取第一页结果（通常50-100条）
    ↓
停止 ❌ 没有翻页逻辑
```

#### 影响和后果

##### 1. 旧集资源可能搜索不到

**场景**：
- PT站按发布时间倒序排列
- 第一页都是最新发布的资源（如131-210集）
- 旧集资源（如1-130集）在后面的页面
- **结果**：无法搜索到旧集资源

**实际案例**：

```bash
【INFO】共搜索到 100 个资源，停止搜索
【INFO】缺失剧集数：[1-92, 94-133, 203-210]
【INFO】匹配完成，共匹配到 0 个资源
```

原因：第一页的100个资源都是最新集数，没有缺失的1-133集。

##### 2. 分段订阅无效

**错误做法**：
```
创建订阅1: 第1-50集
创建订阅2: 第51-100集
```

**为什么无效**：
- 两个订阅都只搜索第一页
- 第一页的结果相同
- 匹配时仍然找不到1-100集

##### 3. 指定集数范围无效

**错误做法**：
```
开始集数: 1
总集数: 50
```

**为什么无效**：
- 搜索阶段仍然只获取第一页
- 第一页没有1-50集的资源
- 匹配失败

#### 为什么这样设计？

##### 性能考虑

- 搜索多页会显著增加耗时
- 大量请求可能触发PT站反爬机制

##### 通常情况够用

- 对于新更新的剧集，最新集数在第一页
- 大多数订阅场景能满足需求

##### 站点排序依赖

- PT站通常按时间倒序
- 最新资源在前，符合订阅需求

---

### 14.2 解决方案

#### 方案1: 手动搜索下载（推荐）

**适用场景**：需要下载旧集或特定集数

**步骤**：

1. **直接访问PT站**
   ```
   登录站点 → 搜索剧名 → 浏览多页找到需要的集数
   ```

2. **手动下载种子**
   - 添加到下载器
   - MoviePilot 会自动识别和整理

3. **保持订阅**
   - 继续自动下载新集

**优点**：
- ✅ 确保能找到资源
- ✅ 下载后自动处理
- ✅ 订阅仍可自动下载新集

#### 方案2: 寻找合集资源

**搜索关键词**：
```
剧名 合集
剧名 Complete
剧名 S01 Complete
剧名 E01-E100
```

**特点**：
- 包含多集甚至整季
- 体积较大
- 一次下载解决多集

#### 方案3: 修改代码添加翻页（根本解决）

**文件**: `app/chain/search.py`

**位置**: Line 386-391

**原代码**:
```python
# 搜索站点
results = self.__search_all_sites(
    mediainfo=mediainfo,
    keyword=search_word,
    sites=sites,
    area=area
) or []
```

**修改为**:
```python
# 搜索站点（支持多页）
results = []
max_pages = 3  # 最多搜索3页，可配置

for page_num in range(0, max_pages):
    page_results = self.__search_all_sites(
        mediainfo=mediainfo,
        keyword=search_word,
        sites=sites,
        area=area,
        page=page_num  # ← 添加页码参数
    ) or []

    if not page_results:
        # 没有结果了，停止翻页
        logger.info(f"第 {page_num} 页无结果，停止搜索")
        break

    results.extend(page_results)
    logger.info(f"第 {page_num} 页获取 {len(page_results)} 个资源")

    # 添加延时，避免触发PT站限制
    if page_num < max_pages - 1:
        import time
        sleep_time = 2  # 延时2秒
        logger.info(f"休眠 {sleep_time} 秒后继续...")
        time.sleep(sleep_time)

logger.info(f"多页搜索完成，共获取 {len(results)} 个资源")
```

**改进建议**：

1. **可配置化**
   ```python
   # 在配置文件中添加
   SEARCH_MAX_PAGES = 3  # 最多搜索页数
   SEARCH_PAGE_DELAY = 2  # 翻页延时（秒）
   ```

2. **添加限制**
   ```python
   max_results = 300  # 最多获取300条
   if len(results) >= max_results:
       logger.info(f"已达到最大结果数 {max_results}，停止搜索")
       break
   ```

3. **智能翻页**
   ```python
   # 只在必要时翻页
   if len(results) < 50:  # 第一页结果少，可能有更多
       # 继续翻页
   ```

**注意事项**：
- ⚠️ 不要设置太多页（建议2-3页）
- ⚠️ 必须添加延时避免触发反爬
- ⚠️ 可能增加搜索耗时
- ⚠️ 部分站点可能限制翻页

#### 方案4: 使用RSS订阅（辅助）

**适用场景**：PT站定期发布补档资源

**原理**：
- 监听PT站的RSS Feed
- 捕获新发布的资源（包括补档）
- 自动下载匹配的资源

**限制**：
- 依赖PT站发布
- 可能需要长时间等待

---

### 14.3 最佳实践建议

#### 对于新剧订阅

**策略**：
```
✅ 直接使用订阅功能
✅ 第一页通常包含最新集数
✅ 自动化效果最好
```

#### 对于补档旧集

**策略**：
```
❌ 不要依赖订阅自动搜索
✅ 手动搜索PT站
✅ 寻找合集资源
✅ 或修改代码添加翻页
```

#### 对于长期追剧

**策略**：
```
✅ 订阅：自动下载新集
✅ 手动：补充缺失的旧集
✅ 结合使用效果最佳
```

---

### 14.4 未来改进方向

#### 1. 配置化翻页功能

```yaml
# config.yaml
search:
  max_pages: 3        # 最多搜索页数
  page_delay: 2       # 翻页延时（秒）
  max_results: 300    # 最多结果数
  enable_paging: true # 是否启用翻页
```

#### 2. 智能翻页策略

```python
# 根据结果数量决定是否翻页
if len(first_page_results) < threshold:
    # 结果太少，继续翻页
elif 匹配到所需资源:
    # 已找到，停止翻页
```

#### 3. 站点分页支持检测

```python
# 自动检测站点是否支持分页
# 避免向不支持的站点发送翻页请求
```

---

**文档版本**: v1.1
**最后更新**: 2025-12-20
**代码版本**: MoviePilot v2
**更新内容**: 添加搜索限制说明和解决方案
