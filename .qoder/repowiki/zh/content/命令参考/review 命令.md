# review 命令

<cite>
**本文档中引用的文件**  
- [cli.rs](file://codex-rs/exec/src/cli.rs)
- [lib.rs](file://codex-rs/exec/src/lib.rs)
- [review_prompts.rs](file://codex-rs/core/src/review_prompts.rs)
- [review_format.rs](file://codex-rs/core/src/review_format.rs)
- [review.rs](file://codex-rs/core/src/tasks/review.rs)
- [protocol.rs](file://codex-rs/protocol/src/protocol.rs)
- [review_prompt.md](file://codex-rs/core/review_prompt.md)
</cite>

## 目录
1. [简介](#简介)
2. [核心功能](#核心功能)
3. [支持的选项](#支持的选项)
4. [典型使用场景](#典型使用场景)
5. [模板系统与分析模型](#模板系统与分析模型)
6. [交互流程](#交互流程)
7. [输出格式](#输出格式)

## 简介

`review` 命令是 Codex 工具中的一个核心功能，用于对现有代码进行质量审查、漏洞检测和优化建议。该命令能够分析代码变更，识别潜在问题，并提供可操作的改进建议。它支持多种审查模式，包括对未提交的更改、特定提交或基础分支的审查。

**Section sources**
- [cli.rs](file://codex-rs/exec/src/cli.rs#L95-L97)

## 核心功能

`review` 命令的主要功能包括：
- **质量审查**：分析代码变更，识别潜在的质量问题。
- **漏洞检测**：检测代码中的安全漏洞和性能问题。
- **优化建议**：提供具体的优化建议，帮助开发者改进代码。

该命令通过分析代码的上下文和变更，生成详细的审查报告，帮助开发者快速定位和解决问题。

**Section sources**
- [review_prompts.rs](file://codex-rs/core/src/review_prompts.rs#L22-L68)

## 支持的选项

`review` 命令支持以下选项：

- `--uncommitted`：审查暂存、未暂存和未跟踪的更改。
- `--base <BRANCH>`：针对指定的基础分支进行审查。
- `--commit <SHA>`：审查由指定提交引入的更改。
- `--title <TITLE>`：为审查摘要中显示的提交提供可选标题。
- `--prompt <PROMPT>`：自定义审查说明。如果使用 `-`，则从标准输入读取。

这些选项可以组合使用，以满足不同的审查需求。

**Section sources**
- [cli.rs](file://codex-rs/exec/src/cli.rs#L116-L148)

## 典型使用场景

### 审查整个目录

```bash
codex review --uncommitted
```

此命令将审查当前目录中的所有未提交更改，包括暂存、未暂存和未跟踪的文件。

### 审查特定文件

```bash
codex review --commit abc1234 --title "Add new feature"
```

此命令将审查由提交 `abc1234` 引入的更改，并在审查摘要中显示标题 "Add new feature"。

### 针对基础分支进行审查

```bash
codex review --base main
```

此命令将审查当前分支与 `main` 分支之间的差异，帮助开发者了解合并请求的影响。

**Section sources**
- [lib.rs](file://codex-rs/exec/src/lib.rs#L552-L601)

## 模板系统与分析模型

`review` 命令利用模板系统和分析模型生成反馈。模板系统定义了审查的指导原则和输出格式，而分析模型则负责实际的代码分析。

### 模板系统

模板系统包含一系列指导原则，用于确定代码中的问题是否需要标记。这些原则包括：
- 问题是否显著影响代码的准确性、性能、安全性或可维护性。
- 问题是否具体且可操作。
- 修复问题所需的严谨性是否与代码库的其他部分一致。

### 分析模型

分析模型根据模板系统的指导原则，对代码进行深入分析。它能够识别代码中的潜在问题，并生成详细的审查报告。

**Section sources**
- [review_prompt.md](file://codex-rs/core/review_prompt.md#L1-L88)

## 交互流程

`review` 命令的交互流程如下：
1. 用户通过命令行或用户界面启动 `review` 命令。
2. 命令解析用户提供的选项，并构建审查请求。
3. 审查请求被发送到分析模型，模型开始分析代码。
4. 分析模型生成审查报告，并将其返回给用户。
5. 用户可以根据审查报告中的建议进行代码改进。

**Section sources**
- [review.rs](file://codex-rs/core/src/tasks/review.rs#L1-L244)

## 输出格式

`review` 命令的输出格式为 JSON，包含以下字段：
- `findings`：审查发现的列表，每个发现包含标题、描述、置信度分数、优先级和代码位置。
- `overall_correctness`：补丁是否正确的总体判断。
- `overall_explanation`：总体判断的解释。
- `overall_confidence_score`：总体判断的置信度分数。

```json
{
  "findings": [
    {
      "title": "<≤ 80 个字符，命令式>",
      "body": "<有效的 Markdown，解释为什么这是一个问题>",
      "confidence_score": <0.0-1.0 的浮点数>,
      "priority": <0-3 的整数，可选>,
      "code_location": {
        "absolute_file_path": "<文件路径>",
        "line_range": {"start": <整数>, "end": <整数>}
      }
    }
  ],
  "overall_correctness": "补丁正确" | "补丁不正确",
  "overall_explanation": "<1-3 句话的解释，证明 overall_correctness 判断>",
  "overall_confidence_score": <0.0-1.0 的浮点数>
}
```

**Section sources**
- [review_format.rs](file://codex-rs/core/src/review_format.rs#L1-L83)