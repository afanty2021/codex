# Codex 测试策略

> 更新时间：2025-12-15 14:56:36
>
本文档汇总了 Codex 项目的整体测试策略、工具配置和最佳实践。

## 测试架构概览

```
测试层次
├── 单元测试 (Unit Tests)
│   ├── Rust: cargo test / cargo nextest
│   └── TypeScript: jest / vitest
├── 集成测试 (Integration Tests)
│   ├── API 端到端测试
│   ├── MCP 服务器测试
│   └── 跨模块交互测试
├── 性能测试 (Performance Tests)
│   ├── 基准测试
│   ├── 内存泄漏检测
│   └── 并发压力测试
└── 端到端测试 (E2E Tests)
    ├── CLI 工作流测试
    ├── TUI 交互测试
    └── 真实场景模拟
```

## 测试工具链

### Rust 测试工具

#### cargo-nextest
并行测试运行器，提供更好的性能和输出格式。

配置文件：`codex-rs/.config/nextest.toml`
```toml
[profile.default]
slow-timeout = { period = "15s", terminate-after = 2 }

# 特定测试的慢速测试配置
[[profile.default.overrides]]
filter = 'test(rmcp_client)'
slow-timeout = { period = "1m", terminate-after = 4 }
```

使用方法：
```bash
# 运行所有测试
cargo nextest run

# 运行特定模块
cargo nextest run -p codex-core

# 运行特定测试模式
cargo nextest run --filter 'integration'

# 保存测试结果
cargo nextest run --archive-file test-results.tar.gz
```

#### cargo-deny
依赖审计和许可证检查：
```bash
# 检查许可证合规性
cargo deny check licenses

# 检查安全漏洞
cargo deny check bans

# 检查所有配置
cargo deny check all
```

### TypeScript 测试工具

#### Jest
用于 shell-tool-mcp 和 sdk/typescript 模块。

配置文件：`jest.config.cjs`
```javascript
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  extensionsToTreatAsEsm: ['.ts'],
  globals: {
    'ts-jest': {
      useESM: true,
    },
  },
  collectCoverageFrom: [
    'src/**/*.ts',
    '!src/**/*.d.ts',
  ],
  coverageThreshold: {
    global: {
      branches: 80,
      functions: 80,
      lines: 80,
      statements: 80,
    },
  },
};
```

## 测试分类和策略

### 1. 单元测试

#### Rust 单元测试
```rust
#[cfg(test)]
mod tests {
    use super::*;
    use pretty_assertions::assert_eq;

    #[test]
    fn test_function_name() {
        let result = function_under_test();
        assert_eq!(result, expected_value);
    }

    #[tokio::test]
    async fn test_async_function() {
        let result = async_function().await;
        assert!(result.is_ok());
    }
}
```

#### TypeScript 单元测试
```typescript
import { functionUnderTest } from '../src/module';

describe('functionUnderTest', () => {
  it('should return expected result', () => {
    const result = functionUnderTest(input);
    expect(result).toBe(expectedOutput);
  });

  it('should handle errors gracefully', () => {
    expect(() => functionUnderTest(invalidInput)).toThrow();
  });
});
```

### 2. 集成测试

#### API 集成测试
```rust
// tests/api_integration.rs
use reqwest::Client;

#[tokio::test]
async fn test_api_endpoint() {
    let client = Client::new();
    let response = client
        .get("http://localhost:3000/api/health")
        .send()
        .await
        .expect("Failed to send request");

    assert_eq!(response.status(), 200);
}
```

### 3. 属性测试（Property Testing）

使用 `proptest` 进行随机测试：
```rust
use proptest::prelude::*;

proptest! {
    #[test]
    fn test_roundtrip(s in "[a-zA-Z0-9]{1,100}") {
        let serialized = serialize(&s);
        let deserialized = deserialize(&serialized);
        assert_eq!(s, deserialized);
    }
}
```

### 4. Mock 和测试替身

#### Mock 服务器
```rust
use mockito::{mock, Server};

#[tokio::test]
async fn test_with_mock_server() {
    let mut server = Server::new();
    let mock = server.mock("GET", "/api/data")
        .with_status(200)
        .with_body(r#"{"value": 42}"#)
        .create();

    let client = Client::new();
    let response = client
        .get(&server.url())
        .send()
        .await
        .unwrap();

    mock.assert();
    assert_eq!(response.status(), 200);
}
```

## 测试覆盖率

### Rust 覆盖率
```bash
# 安装工具
cargo install cargo-tarpaulin

# 生成覆盖率报告
cargo tarpaulin --out Html --output-dir coverage/

# 行覆盖率报告
cargo tarpaulin --line
```

### TypeScript 覆盖率
Jest 内置覆盖率支持：
```bash
# 生成覆盖率报告
npm test -- --coverage

# 查看阈值
npm test -- --coverage --coverageThreshold='{"global":{"branches":80}}'
```

## 性能测试

### 基准测试
```rust
use criterion::{criterion_group, criterion_main, Criterion};

fn benchmark_function(c: &mut Criterion) {
    c.bench_function("function_name", |b| {
        b.iter(|| {
            function_to_benchmark()
        })
    });
}

criterion_group!(benches, benchmark_function);
criterion_main!(benches);
```

运行基准测试：
```bash
cargo bench

# 生成报告
cargo bench -- --output-format html
```

## 测试数据管理

### 测试夹具
```rust
// tests/fixtures/mod.rs
pub fn load_test_data(filename: &str) -> String {
    let path = Path::new("tests/fixtures").join(filename);
    std::fs::read_to_string(path).unwrap()
}

// 使用示例
let test_data = load_test_data("sample.json");
```

### 临时目录管理
```rust
use tempfile::TempDir;

#[test]
fn test_with_temp_dir() {
    let temp_dir = TempDir::new().unwrap();
    let file_path = temp_dir.path().join("test.txt");

    // 测试逻辑...

    // TempDir 会自动清理
}
```

## CI/CD 集成

### GitHub Actions 配置
```yaml
name: Tests

on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    strategy:
      matrix:
        rust: [stable, beta]
    steps:
    - uses: actions/checkout@v3
    - uses: actions-rs/toolchain@v1
      with:
        toolchain: ${{ matrix.rust }}
    - name: Run tests
      run: cargo nextest run
    - name: Check coverage
      run: cargo tarpaulin --ciserver github --output-dir coverage
```

## 特殊测试场景

### 1. 平台特定测试
```rust
#[cfg(target_os = "linux")]
#[test]
fn test_linux_only() {
    // Linux 特定测试
}

#[cfg(target_os = "windows")]
#[test]
fn test_windows_only() {
    // Windows 特定测试
}
```

### 2. 网络依赖测试
```rust
#[tokio::test]
async fn test_network_feature() {
    // 跳过测试如果网络不可用
    if !network_available() {
        return;
    }

    // 网络测试逻辑
}
```

### 3. 资源限制测试
```rust
#[test]
fn test_memory_usage() {
    // 使用内存限制的测试
    let limit = MemoryLimit::new(1024 * 1024); // 1MB

    with_limit(limit, || {
        // 在限制内运行的代码
    });
}
```

## 测试最佳实践

### 1. 命名规范
- Rust: `test_<module>_<feature>_<scenario>`
- TypeScript: `should<ExpectedBehavior>When<StateUnderTest>`

### 2. 测试隔离
- 每个测试独立运行
- 使用临时目录和数据库
- 避免测试间共享状态

### 3. 断言选择
- Rust: 使用 `assert_eq!`, `assert!` 和 `pretty_assertions`
- TypeScript: 使用 Jest 的匹配器

### 4. 异步测试
- Rust: 使用 `#[tokio::test]`
- TypeScript: 使用 async/await 和 done 模式

## 测试数据生成

### Factory 模式
```rust
// tests/factories/mod.rs
pub struct UserFactory;

impl UserFactory {
    pub fn create() -> User {
        User {
            id: Uuid::new_v4(),
            name: "Test User".to_string(),
            email: "test@example.com".to_string(),
        }
    }

    pub fn with_name(name: &str) -> User {
        let mut user = Self::create();
        user.name = name.to_string();
        user
    }
}
```

## 故障测试和容错

### 错误注入
```rust
#[test]
fn test_error_handling() {
    // 模拟各种错误情况
    let scenarios = vec![
        ErrorScenario::NetworkTimeout,
        ErrorScenario::InvalidResponse,
        ErrorScenario::ServerError,
    ];

    for scenario in scenarios {
        let result = handle_error(scenario);
        assert!(result.is_err());
    }
}
```

## 测试文档

### 测试文档示例
```rust
/// # Example
///
/// ```
/// use mycrate::function;
///
/// let result = function(42);
/// assert_eq!(result, 84);
/// ```
///
/// # Panics
///
/// This function will panic if the input is negative.
///
/// ```should_panic
/// use mycrate::function;
///
/// function(-1); // This will panic
/// ```
pub fn double(x: i32) -> i32 {
    if x < 0 {
        panic!("Input must be non-negative");
    }
    x * 2
}
```

## 持续改进

### 测试指标监控
- 测试覆盖率趋势
- 测试执行时间
- 失败率统计
- 测试质量评估

### 测试维护
- 定期审查测试
- 删除过时测试
- 重构重复代码
- 更新测试数据

## 相关资源

- [Rust 测试文档](https://doc.rust-lang.org/book/ch11-00-testing.html)
- [Jest 文档](https://jestjs.io/docs/getting-started)
- [cargo-nextest 文档](https://nextest.equilateral.space/)
- [Tarpaulin 文档](https://github.com/xd009642/tarpaulin)