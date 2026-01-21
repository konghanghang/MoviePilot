# Claude 项目规范

## 重要规则

### 不要修改代码
- 这是别人的开源项目（MoviePilot）
- AI 助手只负责排查问题、分析问题
- **禁止**对代码进行任何修改、编辑或写入操作
- 只能使用 Read、Grep、Glob 等工具进行代码分析和问题排查

## 允许的操作
- 阅读代码文件
- 搜索代码内容
- 分析问题原因
- 提供解决方案建议
- 回答问题

## 禁止的操作
- 修改源代码文件
- 创建新的代码文件
- 使用 Edit、Write 工具修改项目代码
- 提交代码更改

## 用户配置文件位置

### 规则配置文件
- **docs/rule.json** - 用户当前配置的过滤规则
- **docs/rule-group.json** - 用户当前配置的规则组

### 排查问题流程
1. 加载 docs 目录下的 rule.json 和 rule-group.json
2. 根据用户提供的搜索日志
3. 对照规则配置分析问题原因
4. 提供问题诊断和解决方案建议

### 参考文档
详细的技术分析文档位于 docs 目录：
- **docs/subscription-workflow-analysis.md** - 订阅系统完整流程和代码分析
- **docs/subscription-and-notification-guide.md** - 配置指南和常见问题
- **docs/opus-audio-codec.md** - OPUS 音频格式技术说明

---

## 订阅问题排查指引（给 AI）

### 排查订阅过滤问题的标准流程

当用户反馈"订阅搜索不到资源"或"资源都被过滤"时：

**第1步：读取用户配置**
```
1. 读取 docs/rule.json - 查看过滤规则定义
2. 读取 docs/rule-group.json - 查看规则组配置
```

**第2步：分析日志**
```
用户会提供类似这样的日志：
【DEBUG】种子 XXX - 标题 不匹配 filterGlobal 过滤规则
【DEBUG】种子 XXX - 标题 包含 OPUS

关键信息：
- 哪个规则过滤了种子（如 filterGlobal）
- 日志中提示的匹配关键词
```

**第3步：定位问题**
```
在 rule.json 中找到对应规则的 exclude 字段
检查是否有子串误匹配问题：
- OPUS 误匹配 Octopus
- DV 误匹配 ADVERTISE
- DVD 误匹配其他包含DVD的词
```

**第4步：解释原因**
```
向用户解释：
1. 具体是哪个关键词导致过滤
2. 为什么会误匹配（子串匹配问题）
3. 这个关键词的作用（参考 docs/opus-audio-codec.md）
```

**第5步：提供解决方案**
```
建议用户修改 docs/rule.json：
- 使用单词边界：OPUS 改为 \\bOPUS\\b
- 或移除该关键词（如果不需要过滤）
- 提醒 JSON 中需要双反斜杠转义
```

### 关键知识点

**过滤器匹配内容**
- 代码：`app/modules/filter/__init__.py:297`
- 匹配：`title + description + labels`
- 注意：labels 在日志中不显示，但会参与匹配

**常见误匹配案例**
- `OPUS` 匹配 `Octopus`（章鱼）
- `DV` 可能匹配任何包含"DV"的词
- `DVD` 作为子串会匹配很多内容

**正则表达式修复**
- JSON 中使用 `\\b` 表示单词边界（双反斜杠）
- `\\bOPUS\\b` 只匹配完整单词 OPUS
- 不会匹配 Octopus 中的 opus

**搜索限制**
- 代码：`app/chain/search.py:386-391`
- 只搜索第一页（page=0），无翻页
- 旧集资源可能因此搜不到

### 快速参考

**关键代码位置**
- 过滤逻辑：`app/modules/filter/__init__.py:282-376`
- 搜索流程：`app/chain/search.py`
- 订阅管理：`app/chain/subscribe.py`

**用户配置位置**
- 过滤规则：`docs/rule.json`
- 规则组：`docs/rule-group.json`

**详细文档**
- 完整流程：`docs/subscription-workflow-analysis.md`
- 配置指南：`docs/subscription-and-notification-guide.md`
