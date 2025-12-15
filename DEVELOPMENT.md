# OpenAI Codex 开发指南

> 更新时间：2025-12-15 15:06:43
>
> 本文档为 OpenAI Codex 项目的开发人员提供全面的开发指南

## 项目概览

OpenAI Codex 是一个 AI 驱动的代码助手，包含以下主要组件：

- **codex-rs**: Rust 后端核心（48+ 个 crates）
- **codex-cli**: Node.js CLI 封装
- **shell-tool-mcp**: MCP shell 工具服务器
- **sdk/typescript**: TypeScript SDK

## 开发环境设置

### 前置要求

- **Rust**: 1.90+（通过 `rust-toolchain.toml` 自动管理）
- **Node.js**: 22+（通过 `.nvmrc` 推荐）
- **pnpm**: 包管理器
- **Python**: 3.11+（用于构建脚本）

### 快速开始

```bash
# 克隆仓库
git clone https://github.com/openai/codex.git
cd codex

# 安装 Node.js 依赖
pnpm install

# 构建 Rust 组件
cd codex-rs
cargo build

# 运行测试
cargo nextest run
```

## 构建系统

### Rust 构建配置

#### 工具链
```toml
# rust-toolchain.toml
[toolchain]
channel = "1.90"
components = ["rustfmt", "clippy"]
targets = [
    "x86_64-unknown-linux-gnu",
    "aarch64-unknown-linux-gnu",
    "x86_64-apple-darwin",
    "aarch64-apple-darwin",
    "x86_64-pc-windows-msvc"
]
```

#### 代码格式化
```toml
# rustfmt.toml
edition = "2024"
imports_granularity = "Item"
```

#### Clippy 配置
```toml
# clippy.toml
cognitive-complexity-threshold = 30
too-many-arguments-threshold = 8
type-complexity-threshold = 250
```

### Node.js 构建配置

#### 工作空间
```yaml
# pnpm-workspace.yaml
packages:
  - 'codex-cli'
  - 'shell-tool-mcp'
  - 'sdk/typescript'
```

#### 构建脚本
```json
{
  "scripts": {
    "build": "pnpm -r build",
    "test": "pnpm -r test",
    "format": "prettier --write .",
    "format:check": "prettier --check .",
    "lint": "eslint . --ext ts,js",
    "typecheck": "tsc --noEmit"
  }
}
```

## CI/CD 流程

### GitHub Actions 工作流

#### 主要工作流

1. **ci.yml** - Node.js 项目 CI
   - 格式检查（Prettier）
   - README TOC 检查
   - ASCII 字符检查
   - NPM 包构建

2. **rust-ci.yml** - Rust 项目 CI
   - 多平台构建（Linux, macOS, Windows）
   - 多架构支持（x86_64, aarch64）
   - Clippy 检查
   - 单元测试（cargo-nextest）
   - 缓存优化（sccache）

3. **rust-release.yml** - 自动发布
   - 多平台二进制构建
   - GitHub Releases
   - NPM 包发布

4. **sdk.yml** - SDK 专用 CI
   - TypeScript 编译
   - 单元测试
   - 示例代码验证

### 构建优化

#### 缓存策略
- **Cargo Home 缓存**: 缓存编译器和依赖
- **sccache**: 增量编译缓存
- **APT 缓存**: Linux 系统依赖缓存

#### 并行化
- 矩阵构建支持多平台
- cargo-nextest 并行测试
- GitHub Actions 并行作业

## 测试策略

### Rust 测试

#### 测试框架
- **cargo-nextest**: 主要测试运行器
- **cargo test**: 兼容性备用
- **insta**: 快照测试
- **proptest**: 属性测试

#### 测试组织
```
codex-rs/
├── tests/
│   ├── all.rs              # 测试入口
│   └── common/             # 测试工具
└── [crate]/
    └── tests/
        ├── all.rs          # Crate 测试入口
        ├── suite/          # 测试套件
        └── common/         # 测试工具
```

#### 测试配置
```toml
# .config/nextest.toml
[profile.ci-test]
retries = 0
test-threads = 8
slow-timeout = "30s"
```

### TypeScript 测试

#### 测试工具
- **Jest**: 测试框架
- **ts-jest**: TypeScript 支持
- **@types/jest**: 类型定义

#### 测试命令
```bash
# 运行所有测试
pnpm test

# 监视模式
pnpm test:watch

# 覆盖率
pnpm test:coverage
```

## 代码质量

### Rust 代码质量

#### 工具集成
- **rustfmt**: 自动代码格式化
- **clippy**: Linting 和最佳实践
- **cargo-deny**: 依赖审计和许可证检查
- **cargo-shear**: 未使用依赖检测

#### 依赖管理
```toml
# deny.toml 配置
[advisories]
# 安全漏洞检测

[licenses]
# 许可证合规性
allow = ["MIT", "Apache-2.0", "BSD-3-Clause"]

[bans]
# 重复版本检测
multiple-versions = "warn"
```

### TypeScript 代码质量

#### ESLint 配置
```javascript
// eslint.config.js
module.exports = {
  extends: [
    '@typescript-eslint/recommended',
    'prettier'
  ],
  rules: {
    // 自定义规则
  }
};
```

#### Prettier 配置
```toml
# .prettierrc.toml
tab_width = 2
use_tabs = false
print_width = 100
trailing_comma = "es5"
```

## 发布流程

### 版本管理

#### 语义化版本
- 遵循 SemVer 2.0.0
- 自动化 changelog 生成（git-cliff）

#### 发布步骤

1. **Rust 二进制发布**
   ```bash
   # 创建发布标签
   git tag v0.40.0
   git push origin v0.40.0

   # 触发 GitHub Actions 自动发布
   ```

2. **NPM 包发布**
   ```bash
   # 构建并发布
   pnpm run build:npm
   pnpm publish
   ```

### 发布配置

#### Changelog 配置
```toml
# cliff.toml
[changelog]
header = """
# Changelog
"""

[git]
conventional_commits = true

commit_parsers = [
  { message = "^feat", group = "🚀 Features" },
  { message = "^fix", group = "🪲 Bug Fixes" },
  { message = "^bump", group = "🛳️ Release" }
]
```

## 开发工具

### IDE 配置

#### VS Code
- **Rust Analyzer**: Rust 语言服务器
- **TypeScript TSC**: TypeScript 支持
- **ESLint**: 代码质量检查
- **Prettier**: 代码格式化

#### 推荐扩展
```json
{
  "recommendations": [
    "rust-lang.rust-analyzer",
    "ms-vscode.vscode-typescript-next",
    "esbenp.prettier-vscode",
    "ms-vscode.vscode-eslint"
  ]
}
```

### 调试配置

#### Rust 调试
```json
{
  "type": "lldb",
  "request": "launch",
  "name": "Debug Rust",
  "cargo": {
    "args": ["build", "--bin=codex"],
    "filter": {
      "name": "codex",
      "kind": "bin"
    }
  }
}
```

#### TypeScript 调试
```json
{
  "type": "node",
  "request": "launch",
  "name": "Debug TypeScript",
  "program": "${workspaceFolder}/dist/index.js",
  "preLaunchTask": "tsc: build"
}
```

## 性能优化

### Rust 性能优化

#### 编译优化
- **Profile-Guided Optimization (PGO)**
- **Link-Time Optimization (LTO)**
- **codegen-units** 调优

#### 运行时优化
- **异步并发** (tokio)
- **零拷贝操作**
- **内存池管理**
- **流式处理**

### TypeScript 性能优化

#### 构建优化
- **swc** 替代 tsc
- **Tree-shaking**
- **代码分割**

#### 运行时优化
- **事件循环优化**
- **内存泄漏防护
- **异步 I/O**

## 安全最佳实践

### Rust 安全

#### 内存安全
- 避免不安全代码块
- 使用类型系统保证安全
- 定期运行 `cargo-audit`

#### 依赖安全
```bash
# 检查安全漏洞
cargo audit

# 检查许可证合规
cargo deny check licenses
```

### TypeScript 安全

#### 输入验证
- 使用 Zod 进行运行时类型检查
- 验证所有外部输入
- 防止注入攻击

#### 依赖安全
```bash
# 检查漏洞
pnpm audit

# 自动修复
pnpm audit --fix
```

## 故障排查

### 常见问题

1. **构建失败**
   - 检查 Rust 工具链版本
   - 清理缓存：`cargo clean`
   - 更新依赖：`cargo update`

2. **测试失败**
   - 检查环境变量
   - 确保测试数据存在
   - 运行单个测试：`cargo test test_name`

3. **发布失败**
   - 验证版本号格式
   - 检查 changelog
   - 确认权限设置

### 调试技巧

#### Rust 调试
- 使用 `dbg!` 宏快速调试
- 启用详细日志：`RUST_LOG=debug`
- 使用 `cargo-expand` 查看宏展开

#### TypeScript 调试
- 使用 `debugger` 语句
- 配置 source maps
- 使用 VS Code 调试器

## 贡献指南

### 提交代码

1. **Fork 仓库**
2. **创建功能分支**
   ```bash
   git checkout -b feature/new-feature
   ```
3. **提交更改**
   ```bash
   git commit -m "feat: add new feature"
   ```
4. **推送并创建 PR**

### 代码审查

- 自动化检查必须通过
- 至少一个审查者批准
- 遵循代码风格指南

### 文档更新

- 更新相关 README
- 添加代码注释
- 更新 changelog

## 资源链接

- [项目仓库](https://github.com/openai/codex)
- [文档站点](https://docs.anthropic.com)
- [Discord 社区](https://discord.gg)
- [问题跟踪](https://github.com/openai/codex/issues)

## 变更记录

### 2025-12-15 15:06:43
- ✨ 创建开发指南文档
- 📝 详细说明构建和测试流程
- 🔧 添加代码质量工具配置
- 🏗️ 记录发布和故障排查流程