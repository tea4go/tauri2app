# tauri2app

> Tauri 2 跨平台桌面应用 — 前端 **Vue 3 + TypeScript + Vite**，后端 **Rust**

[![Tauri](https://img.shields.io/badge/Tauri-2.x-FFC131?logo=tauri)](https://tauri.app)
[![Vue](https://img.shields.io/badge/Vue-3.x-4FC08D?logo=vue.js)](https://vuejs.org)
[![Vite](https://img.shields.io/badge/Vite-6.x-646CFF?logo=vite)](https://vite.dev)
[![Rust](https://img.shields.io/badge/Rust-2021%20edition-000000?logo=rust)](https://www.rust-lang.org)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

---

## 目录

- [项目简介](#项目简介)
- [技术栈](#技术栈)
- [快速开始](#快速开始)
- [常用命令](#常用命令)
- [项目结构](#项目结构)
- [架构要点](#架构要点)
- [License](#license)

## 项目简介

tauri2app 是一个基于 **Tauri 2** 的跨平台桌面应用，具备以下特性：

- 使用 **Vue 3 Composition API**（`<script setup>` + TypeScript）构建前端界面
- 使用 **Rust** 编写高性能的后端逻辑
- 通过 Tauri IPC（`invoke`）实现前后端双向通信
- 支持 **Windows / macOS / Linux** 桌面环境，并可通过 Tauri 2 打包移动端应用（Android / iOS）

## 技术栈

| 层级 | 技术 | 说明 |
|------|------|------|
| 框架 | Tauri 2 | 桌面壳 + Rust 后端运行时 |
| 前端 | Vue 3 + TypeScript | Composition API + SFC |
| 构建 | Vite 6 | 开发服务器与 HMR |
| 后端 | Rust (edition 2021) | 系统调用与业务逻辑 |
| 包管理 | pnpm | 依赖管理与脚本执行 |
| IPC | `@tauri-apps/api` | 前后端调用桥接 |

## 快速开始

### 前置要求

- **Node.js** >= 18
- **pnpm**（推荐）或 npm / yarn
- **Rust** 工具链（rustup + cargo）
- 平台依赖：参见 [Tauri 官方 Prerequisites](https://v2.tauri.app/start/prerequisites/)

### 安装依赖

```bash
pnpm install
```

### 启动开发模式

```bash
pnpm tauri dev
```

这会自动先启动 Vite 开发服务器（`http://localhost:1420`），再启动 Rust 桌面窗口并连接到前端。

## 常用命令

```bash
# 开发（启动完整 Tauri 应用）
pnpm tauri dev

# 仅启动前端 Vite（端口 1420，不可变更）
pnpm dev

# 构建桌面安装包（先类型检查 + 前端构建，再打包）
pnpm tauri build

# 仅前端构建 + 类型检查
pnpm build

# 预览前端构建产物
pnpm preview

# Tauri CLI 帮助
pnpm tauri --help

# 移动端（需先初始化）
pnpm tauri android dev
pnpm tauri ios dev
```

## 项目结构

```
tauri2app/
├── public/               # 静态资源
│   ├── tauri.svg
│   └── vite.svg
├── src/                  # 前端源码（Vue 3 + TS）
│   ├── assets/
│   │   └── vue.svg
│   ├── App.vue           # 根组件
│   ├── main.ts           # Vue 应用入口
│   └── vite-env.d.ts     # Vite 环境类型声明
├── src-tauri/            # Rust 后端
│   ├── capabilities/     # Tauri 2 权限声明
│   │   └── default.json
│   ├── gen/              # Tauri 自动生成（schema 等）
│   ├── icons/            # 应用图标
│   ├── src/
│   │   ├── lib.rs        # 核心逻辑（run 入口、command 注册）
│   │   └── main.rs       # 二进制入口
│   ├── Cargo.toml        # Rust 依赖声明
│   ├── build.rs          # Tauri 构建脚本
│   └── tauri.conf.json   # Tauri 应用配置
├── index.html            # HTML 入口
├── package.json          # 前端依赖与脚本
├── tsconfig.json         # TypeScript 配置
├── vite.config.ts        # Vite 配置
├── CLAUDE.md             # Claude Code 项目指引
└── LICENSE               # MIT 许可证
```

## 架构要点

### 前后端分离

- **前端**（`src/`）：Vue 3 SFC，由 Vite 在 `localhost:1420` 热重载
- **后端**（`src-tauri/src/`）：Rust crate，`main.rs` 为 bin 入口，逻辑集中在 `lib.rs` 的 `run()` 函数
- **Rust crate 命名**：`tauri2app_lib`（加 `_lib` 后缀以避免 Windows 上 cdylib 与 bin 同名冲突）

### IPC 通信

前端通过 `@tauri-apps/api` 的 `invoke("命令名", { 参数 })` 调用 Rust 命令；后端用 `#[tauri::command]` 定义并注册到 `invoke_handler!` 宏。

示例：

```typescript
// 前端调用
const msg = await invoke("greet", { name: "World" });
```

```rust
// Rust 命令
#[tauri::command]
fn greet(name: &str) -> String {
    format!("Hello, {}! You've been greeted from Rust!", name)
}
```

### 权限模型（Capabilities）

`src-tauri/capabilities/default.json` 声明窗口可用权限。当前已开启 `core:default` 和 `opener:default`。

**新增插件或调用受限 API 时，必须在此文件追加对应权限**，否则前端调用会被运行时拒绝。

### 配置耦合点

- 端口 **1420** 在 `vite.config.ts`（`server.port`、`strictPort: true`）和 `tauri.conf.json`（`build.devUrl`）两处必须保持一致
- `tauri.conf.json` 中 `frontendDist: "../dist"` 指向 Vite 构建产物
- `identifier`（`com.admin.tauri2app`）用于打包，发布前应改为正式域名反写
- `vite.config.ts` 显式忽略 `src-tauri/**`，Rust 改动不会触发 Vite 热重载

### 移动端支持

通过 `#[cfg_attr(mobile, tauri::mobile_entry_point)]` 标注了移动端入口点，可使用 `pnpm tauri android dev` / `pnpm tauri ios dev` 开发（需先运行对应的 `init` 命令）。

## License

本项目基于 **MIT License** 开源 — 详见 [LICENSE](./LICENSE)。
