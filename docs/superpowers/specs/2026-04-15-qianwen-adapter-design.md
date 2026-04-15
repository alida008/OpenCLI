# 千问适配器设计文档

**日期：** 2026-04-15
**状态：** 已批准

## 概述

为 OpenCLI 添加通义千问（qianwen.com）适配器，支持通过命令行与千问对话，返回原始回答 + 要点提取。

## 命令设计

### ask 命令

```bash
opencli qianwen ask <问题> [--group <分组名>] [--subgroup <子对话名>] [--new] [--raw|--summary|--json]
```

### 参数说明

| 参数 | 必需 | 说明 |
|------|------|------|
| `<问题>` | 是 | 要问千问的问题 |
| `--group <分组名>` | 否 | 指定分组，不传则沿用上次使用的分组 |
| `--subgroup <子对话名>` | 否 | 显式指定子对话，不传则自动判断是否沿用 |
| `--new` | 否 | 强制创建新子对话 |
| `--raw` | 否 | 仅输出原始回答 |
| `--summary` | 否 | 仅输出要点提取 |
| `--json` | 否 | 输出 JSON 格式 |

### 行为逻辑

| 调用方式 | Group 行为 | Subgroup 行为 |
|----------|-----------|---------------|
| `ask <问题>` | 沿用上次 group | 自动判断（相关则沿用，否则新建） |
| `ask <问题> --group X` | 切换到 X | 沿用 X 中最近的 subgroup |
| `ask <问题> --subgroup Y` | 沿用上次 group | 切换到 Y（不存在则报错） |
| `ask <问题> --new` | 沿用上次 group | 创建新 subgroup |
| `ask <问题> --group X --new` | 切换到 X | 创建新 subgroup |

### list 命令

```bash
opencli qianwen list
```

列出所有可用分组，用于发现和选择。

## 架构

### 文件结构

```
clis/qianwen/
├── ask.js                    # ask 命令实现
├── utils.js                  # 通用工具函数
├── state.json                # 状态存储（last_group, last_subgroup）
└── utils.test.js             # 单元测试
```

### 核心模块

```
┌─────────────────────────────────────────────────────────┐
│                    ask.js                               │
│  - 命令入口                                             │
│  - 参数解析                                             │
│  - 调用 utils 执行流程                                  │
├─────────────────────────────────────────────────────────┤
│                    utils.js                             │
│  - navigateToQianwen()     # 访问千问网页               │
│  - selectGroup()           # 选择/创建分组              │
│  - selectSubgroup()        # 选择/创建子对话            │
│  - sendQuestion()          # 发送问题                   │
│  - waitForResponse()       # 等待回答                   │
│  - readResponse()          # 提取回答内容               │
│  - extractKeyPoints()      # 要点提取                   │
│  - updateState()           # 更新状态文件               │
│  - loadState()             # 读取状态文件               │
└─────────────────────────────────────────────────────────┘
```

### 数据流

```
用户输入
    ↓
[ask.js] 解析参数
    ↓
[utils.loadState()] 读取上次状态
    ↓
[utils.navigateToQianwen()] 访问 qianwen.com
    ↓
[utils.selectGroup()] 选择/创建分组
    ↓
[utils.selectSubgroup()] 判断是否沿用子对话
    ↓
[utils.sendQuestion()] 发送问题
    ↓
[utils.waitForResponse()] 等待回答
    ↓
[utils.readResponse()] 提取原始回答
    ↓
[utils.extractKeyPoints()] 要点提取
    ↓
[utils.updateState()] 更新状态
    ↓
输出：原始回答 + 要点
```

## 状态管理

### state.json 结构

```json
{
  "last_group": "默认分组",
  "last_group_id": "group_123",
  "last_subgroup_id": "chat_456",
  "last_subgroup_title": "关于 XXX 的讨论",
  "last_update": "2026-04-15T14:30:00Z"
}
```

### 相关性判断算法

```javascript
// 简化版：基于关键词重叠
function calculateRelatedness(newQuestion, historyTitle, historyContent) {
  const newWords = newQuestion.split(/[，。、\s]+/).filter(w => w.length > 1);
  const historyText = historyTitle + historyContent;
  
  const matches = newWords.filter(w => historyText.includes(w)).length;
  return matches / newWords.length; // 0-1 的相关性分数
}

// 阈值：> 0.3 认为相关，沿用子对话
```

## 输出格式

### 默认格式（plain）

```
💬 千问回答：

<千问的原始回答内容>

---
📌 要点提取：
1. <要点 1>
2. <要点 2>
3. <要点 3>
```

### JSON 格式

```json
{
  "original": "<千问原始回答>",
  "keyPoints": ["要点 1", "要点 2", "要点 3"],
  "group": "分组名",
  "subgroup": "子对话名"
}
```

## 错误处理

| 错误场景 | 检测方式 | 处理方式 |
|----------|----------|----------|
| 未登录 | 导航后检测登录页特征（登录框、未登录提示文案） | 返回 77 + 提示「请先在浏览器登录千问」 |
| 分组不存在 | 侧边栏搜索失败 | 自动创建新分组 |
| 子对话不存在（--subgroup 指定） | 列表匹配失败 | 返回错误，提示可用子对话列表 |
| 超时（默认 120s） | 计时器触发 | 返回 `[TIMEOUT] No response within Xs`，退出码 75 |
| 无响应 | 未检测到新消息 DOM | 返回 `[NO_RESPONSE] 未检测到新消息`，退出码 66 |
| state.json 并发冲突 | 文件锁失败 | 等待 100ms 重试，最多 5 次 |

## 退出码

遵循 OpenCLI 标准：
- `0` - 成功
- `66` - 空结果/无响应
- `75` - 超时
- `77` - 未登录

## 实现步骤

1. **UI 探索** - 手动访问千问，记录 DOM 结构（登录页、侧边栏、对话区域）
2. **基础框架** - 创建 ask.js、utils.js、list.js 骨架
3. **导航与登录检测** - 实现 navigateToQianwen + 登录状态检测
4. **状态管理** - 实现 loadState + updateState（带文件锁）
5. **分组管理** - 实现 selectGroup + 分组列表获取（为 list 命令服务）
6. **子对话管理** - 实现 selectSubgroup + 相关性判断
7. **问答流程** - 实现 sendQuestion + waitForResponse + 超时处理
8. **内容提取** - 实现 readResponse + extractKeyPoints
9. **list 命令** - 实现分组列表输出
10. **测试** - 编写单元测试和集成测试
11. **文档** - 添加 README 和使用示例

## 依赖

- 已有：`opencli browser` 基础设施
- 已有：gemini 适配器的参考实现
- 需要：千问网页的 DOM 结构探索

## 测试计划

```javascript
// 单元测试
- loadState / updateState
- calculateRelatedness
- normalizeBooleanFlag

// 集成测试（需要真实浏览器）
- 完整问答流程
- 分组切换
- 子对话沿用逻辑
```

## 后续扩展

- [ ] 支持多模态输入（图片）
- [ ] 支持流式输出
- [ ] 支持模型选择（通义千问不同版本）
- [ ] 支持对话历史导出
