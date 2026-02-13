# Tree-sitter 在项目中的使用说明

本文档详细说明了本项目如何使用 Tree-sitter 进行代码解析，重点介绍其在 Bash 命令安全检测中的应用。

## 1. 概述

本项目引入 Tree-sitter 主要是为了增强 **Bash Shell 命令的安全性和可控性**。通过构建命令的抽象语法树（AST），我们能够更精准地：

*   **识别命令结构**：区分主命令、参数、管道、重定向等。
*   **检测潜在风险**：识别危险的重定向操作、命令注入尝试等。
*   **实施细粒度策略**：允许对复杂命令（如 `git log && echo "done"`）中的每个子命令分别进行权限检查。

目前，Tree-sitter 仅用于 **Bash** 命令的解析。PowerShell 命令的解析采用原生 PowerShell 脚本实现。

## 2. 技术架构

### 核心依赖

项目使用了 Tree-sitter 的 WebAssembly (WASM) 版本，以确保在 Node.js 环境中跨平台的一致性运行，且无需依赖本地编译工具链（如 `node-gyp`）。

*   **`web-tree-sitter`**: Tree-sitter 的 WASM 绑定库。
*   **`tree-sitter-bash`**: Bash 语言的语法定义（Grammar），同样编译为 WASM 格式。

### 构建配置

在 `esbuild.config.js` 中，我们配置了专门的插件来处理 `.wasm` 文件：

```javascript
// esbuild.config.js (部分示意)
function createWasmPlugins() {
  // ... 自定义插件逻辑 ...
  return [wasmBinaryPlugin, wasmLoader({ mode: 'embedded' })];
}
```

*   **Embedded Mode**: `.wasm` 文件被作为二进制数据直接嵌入到打包后的 JavaScript 文件中（或作为资源文件加载），确保分发时文件完整性。
*   **Binary Import**: 代码中通过 `import('.../tree-sitter.wasm?binary')` 的方式显式引入 WASM 二进制数据。

## 3. 实现细节

核心逻辑位于 `packages/core/src/utils/shell-utils.ts`。

### 3.1 初始化 (Initialization)

解析器的初始化是异步的，主要包含两步：

1.  **加载 WASM 二进制文件**：并行加载 `web-tree-sitter` 的主逻辑 WASM 和 Bash 语法的 WASM。
2.  **初始化 Parser**：调用 `Parser.init()` 和 `Language.load()`。

```typescript
// packages/core/src/utils/shell-utils.ts
async function loadBashLanguage(): Promise<void> {
  const [treeSitterBinary, bashBinary] = await Promise.all([
    // 加载 tree-sitter.wasm
    // 加载 tree-sitter-bash.wasm
  ]);
  await Parser.init({ wasmBinary: treeSitterBinary });
  bashLanguage = await Language.load(bashBinary);
}
```

### 3.2 解析与超时控制 (Parsing & Timeout)

为了防止恶意构造的超长或极度复杂的命令导致主线程卡死，解析过程加入了超时控制。

*   **函数**: `parseCommandTree(command, timeoutMicros)`
*   **机制**: 利用 `parser.parse()` 的 `progressCallback` 回调，检查执行时间是否超过阈值（默认 1 秒）。如果超时，解析会被中断并返回 `null`。

### 3.3 AST 遍历与命令提取 (AST Traversal)

解析成功后，通过 `collectCommandDetails` 函数遍历 AST，提取出所有需要执行的子命令。

*   **节点类型识别**: 关注 `command`, `declaration_command`, `unset_command` 等节点。
*   **命令名称提取**: 从节点的子节点中（如 `name` 字段）提取并标准化命令名称。

### 3.4 安全检测 (Security Checks)

Tree-sitter 提供了比正则表达式更可靠的检测能力：

1.  **重定向检测 (`hasRedirection`)**:
    *   检查 AST 中是否存在 `redirected_statement`, `file_redirect`, `heredoc_redirect` 等节点。
    *   如果发现重定向符号（`>`, `<`, `<<` 等），策略引擎可以据此决定是否需要用户额外确认。

2.  **Prompt Injection 检测 (`hasPromptCommandTransform`)**:
    *   检测是否存在特定的 Bash 扩展模式（如 `${@P}`），这可能被用于 Prompt Injection 攻击。

3.  **语法错误检测**:
    *   使用 Tree-sitter 查询语法 `(ERROR) @error (MISSING) @missing` 来快速定位语法错误。如果命令存在语法错误，解析结果会被标记为 `hasError: true`，此时系统可能会回退到更保守的策略。

## 4. 局限性与替代方案

### PowerShell

由于 PowerShell 的语法极其复杂且动态，社区缺乏成熟的 Tree-sitter Grammar。因此，对于 Windows 上的 PowerShell 命令，我们**不使用 Tree-sitter**。

*   **替代方案**: 通过 `spawnSync` 调用一个内嵌的 PowerShell 脚本（Base64 编码）。
*   **原理**: 利用 PowerShell 自带的解析器 `[System.Management.Automation.Language.Parser]` 来分析命令结构。

## 5. 维护指南

### 添加新语言

如果未来需要支持其他 Shell（如 Zsh 或 Fish）：
1.  安装对应的 npm 包（如 `tree-sitter-zsh`）。
2.  在 `shell-utils.ts` 中添加对应的 WASM 加载逻辑。
3.  确保 `esbuild.config.js` 正确处理新的 `.wasm` 文件。

### 更新 Grammar

更新 `tree-sitter-bash` 版本时，需验证：
1.  WASM 文件是否能被正确加载。
2.  现有的节点类型名称（如 `command`, `file_redirect`）是否发生变更。
3.  运行 `packages/core/src/policy/shell-safety.test.ts` 确保解析行为符合预期。
