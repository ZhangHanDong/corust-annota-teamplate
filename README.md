# 从原始问题到标准化问题的转换教程

## 概述

本教程说明如何将原始的技术文档、问题描述或代码片段，通过标准模板和 AI 辅助，转换为符合 CorustEvaluation v2.1 标准的问题文件。

### 转换流程图

```
原始问题文档                    标准模板
(tokio-deadlock.md)  +  (teamplate.md)
        ↓                          ↓
        └─────────→  AI 分析  ←────┘
                        ↓
                   结构化转换
                        ↓
                 标准化问题文件
         (tokio-task-lifecycle-port-conflict.md)
                        ↓
                        提交你的问题标注
```

### 目标输出

- ✅ 结构化问题文件（符合模板格式）
- ✅ 通用化代码示例（去除项目特定细节）
- ✅ 清晰的问题-解决方案对应

---

## 准备工作

### 1. 文件结构

```bash
questions-collection/
├── teamplate.md              # 标准模板（参考）
├── tokio-deadlock.md         # 原始问题示例
├── 说明.md                   # 本教程
└── [你的问题].md             # 待转换的原始问题
```

### 2. 环境配置（使用 AI 辅助）

```bash
# 配置 Claude Code
export ANTHROPIC_USER_EMAIL="your-email@example.com"
export CLAUDE_MODEL="claude-sonnet-4-5-20250929"
export AI_PROVIDER="cc"
```

或者用其他 ai 辅助工具，比如 claude code/codex 等

### 3. 理解模板结构

**必填字段**：

| 字段 | 说明 | 要求 |
|------|------|------|
| **title** | 问题标题 | 一句话，≤50字符，问题导向 |
| **Category** | 技术分类 | 参考 [https://annota.corust.ai/](https://annota.corust.ai/)  |
| **Description** | 问题描述 | 2-3句话，包含问题+影响+原因 |
| **Background** | 背景上下文 | 100-150字，必要的技术背景 |
| **Problem Code/Description** | 问题代码 | 精简的可复现代码 + 注释 |
| **Error Message** | 错误信息 | 编译错误/运行时错误/日志 |
| **Solution** | 解决方案 | 完整代码 + 解释 + 关键点 |
| **Attachments** | 附件 | 可选 |

---

## 详细步骤

### Step 1: 使用 AI 分析原始问题

#### 1.1 启动 Claude Code

```bash
cd /your/questions-collection/path
claude code
```

#### 1.2 发送分析提示词

在 Claude Code 对话中：

```
请你将 questions-collection/tokio-deadlock.md
整理为符合 questions-collection/teamplate.md 格式的新问题。

要求：
1. 提取核心技术问题，去除项目特定细节（如 ChromeLauncher, BrightData 等）
2. 代码示例要通用化和简化，使用通用的结构体/函数名
3. 添加清晰的 ❌/✅ 标记区分错误和正确代码
4. 包含元问题映射信息（meta_code, tech_category, cognitive_level）
5. 新文件名使用描述性命名（kebab-case 格式）
6. 确保代码可编译或明确标记为 compile_fail
```

#### 1.3 AI 分析过程

AI 会自动执行以下步骤：

1. **读取原始问题**：分析 tokio-deadlock.md 内容
2. **读取模板**：理解 teamplate.md 格式要求
3. **提取核心问题**：
   - 问题定义：Fire-and-forget 反模式
   - 根本原因：未保存 JoinHandle
   - 影响：端口冲突、资源泄漏
   - 解决方案：任务生命周期管理

4. **通用化代码**：移除项目特定细节
5. **确定分类**：参考 [https://annota.corust.ai/](https://annota.corust.ai/) 
6. **生成结构化输出**：创建新文件

---

### Step 2: 审查 AI 生成的结果

#### 质量检查清单

**内容质量**：

- [ ] **Title**: 简洁明确，包含关键技术词汇
- [ ] **Category**: 准确分类
- [ ] **Description**: 包含问题定义 + 影响 + 根本原因
- [ ] **Problem Code**: 精简且有明确标记
- [ ] **Solution**: 包含完整代码 + 关键点总结

**代码质量**：

- [ ] 代码可编译或明确标记 compile_fail
- [ ] 使用 ❌ 标记错误模式
- [ ] 使用 ✅ 标记正确模式
- [ ] 移除项目特定依赖
- [ ] 变量名通用化

---

### Step 3: 提交你的问题标注

- 打开 [https://annota.corust.ai/](https://annota.corust.ai/) ，GitHub 登录
- 复制你生成好的问题内容到对应表单并提交


#### 完整示例

#### 输入：原始问题

**文件**: `tokio-deadlock.md`（317 行）
- 包含项目特定细节（ChromeLauncher, BrightData）
- 混合问题描述、解决方案、测试验证

#### 处理：AI 转换提示词

```
请你将 questions-collection/tokio-deadlock.md
整理为符合 questions-collection/teamplate.md 格式的新问题
```

#### 输出：标准化问题

**文件**: `tokio-task-lifecycle-port-conflict.md`（~200 行）

**改进对比**：

| 方面 | 原始文档 | 标准化问题 |
|------|----------|-----------|
| **长度** | 317 行 | 200 行 (-37%) |
| **代码示例** | 项目特定 | 通用化 |
| **结构** | 自由格式 | 模板化 |
| **可复用性** | 低 | 高 |

---


## 质量标准

### 优秀问题的特征

✅ **清晰性**：
- 标题一眼看懂问题
- 描述言简意赅
- 代码最小化可复现

✅ **完整性**：
- 问题 + 原因 + 影响
- 错误信息 + 解决方案
- 元数据标注齐全

✅ **通用性**：
- 不依赖特定项目
- 代码可直接运行
- 适用多种场景

✅ **教育性**：
- 学习目标明确
- 相关概念清晰
- 实际应用场景丰富
