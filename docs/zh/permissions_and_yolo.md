# YOLO 模式与权限管理 (YOLO Mode & Permission Management)

本文档将介绍 Gemini CLI 的权限管理系统，以及如何开启和使用 **YOLO 模式**（即用户所称的 "yoyo 模式"），实现全自动的工具调用而无需手动确认。

## 什么是 YOLO 模式 (yoyo 模式)？

**YOLO** (You Only Live Once) 模式是 Gemini CLI 的一种高权限运行模式。在此模式下，Agent 对所有工具的调用都将被自动批准，无需用户手动确认。

这对于需要进行大量自动化操作、且你完全信任 Agent 行为的场景非常有用。

### 如何开启 YOLO 模式

你可以通过以下两种方式开启 YOLO 模式：

#### 1. 使用快捷键（推荐）

在 Gemini CLI 交互界面中，随时按下快捷键：

`Ctrl + Y`

屏幕右下角或状态栏会出现 "YOLO mode" 字样，表示模式已激活。再次按下即可关闭。

#### 2. 修改配置文件

你可以在用户的 `settings.json` 文件中配置默认开启（注意：出于安全考虑，管理员策略可能会禁用此功能）。

文件路径通常位于：
- **Mac/Linux**: `~/.gemini/settings.json`
- **Windows**: `%USERPROFILE%\.gemini\settings.json`

但通常我们建议通过交互式的快捷键来动态切换。

## Gemini CLI 的权限管理体系

Gemini CLI 拥有一个分层的权限管理系统，确保在灵活性和安全性之间取得平衡。权限检查按照优先级（Priority）从高到低进行匹配。

### 权限层级 (Tiers)

权限策略分为三个层级（Tier），优先级从低到高：

1.  **Default Tier (默认层级)** - Tier 1
    - 包含系统内置的默认策略。
    - 例如：读取文件通常是被允许的，而写入文件或执行 Shell 命令通常需要询问用户（Ask User）。

2.  **User Tier (用户层级)** - Tier 2
    - 包含用户在 `settings.json` 中定义的配置。
    - 包含用户在交互过程中产生的临时授权（Dynamic Rules）。
    - 这里的规则会覆盖默认层级。

3.  **Admin Tier (管理员层级)** - Tier 3
    - 企业级配置，通常位于系统目录。
    - 具有最高优先级，用户无法覆盖。
    - 即使开启 YOLO 模式，如果管理员层级明确禁止了某项操作，该操作仍会被拒绝。

### YOLO 模式在权限体系中的位置

YOLO 模式本质上是在 **User Tier (Tier 2)** 注入了一条高优先级的“允许所有”规则。

- **优先级**: 它的优先级非常高（接近 User Tier 的顶端），因此可以覆盖绝大多数需要 "Ask User" 的默认规则。
- **限制**: 它**不能**覆盖 Admin Tier (Tier 3) 的禁止规则，也不能覆盖用户自己在 Settings 中明确排除（Exclude）的工具。

### 安全注意事项

开启 YOLO 模式意味着你授予了 Agent 极大的权限，包括但不限于：
- 任意读写文件
- 执行任意 Shell 命令
- 安装依赖包

**请务必只在可信的环境中，或者当你时刻盯着屏幕输出时使用 YOLO 模式。**

如果你的企业环境要求更高的安全性，管理员可以通过系统配置禁用 YOLO 模式：

```json
// system-settings.json
{
  "security": {
    "disableYoloMode": true
  }
}
```

一旦禁用，即便用户按下 `Ctrl + Y` 也无法进入 YOLO 模式。

## 总结

- **yoyo 模式** 实为 **YOLO 模式**。
- 快捷键 `Ctrl + Y` 开启。
- 它让所有工具调用自动通过。
- 它的权限优先级高于默认规则，但低于管理员强制策略。
