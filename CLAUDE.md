# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概览

Tauri 2 桌面应用，前端使用 Vue 3 + TypeScript + Vite，后端使用 Rust。包管理器为 **pnpm**。

## 常用命令

```bash
# 开发（启动 Tauri 应用 — 会先调用 pnpm dev 起 Vite，再启动 Rust 进程）
pnpm tauri dev

# 仅启动前端 Vite（端口固定 1420，不可变更，否则 Tauri 连接失败）
pnpm dev

# 构建（先 vue-tsc 类型检查 + vite build，然后打包 Tauri 安装包）
pnpm tauri build

# 仅前端构建 + 类型检查
pnpm build

# 预览前端构建产物
pnpm preview
```

注意：`pnpm tauri dev` / `pnpm tauri build` 内部会通过 `tauri.conf.json` 中的 `beforeDevCommand` / `beforeBuildCommand` 自动调用 `pnpm dev` / `pnpm build`，不需要手动并行启动两端。

移动端入口点已通过 `#[cfg_attr(mobile, tauri::mobile_entry_point)]` 标注，可使用 `pnpm tauri android dev` / `pnpm tauri ios dev`（需先运行 `pnpm tauri android init` 等初始化命令）。

## 架构要点

### 前后端进程模型

- **前端**（`src/`）：Vue 3 SFC（`<script setup>` + TS），由 Vite 在 `localhost:1420` 提供。`vite.config.ts` 显式忽略 `src-tauri/**`，Rust 改动不会触发 Vite 热重载。
- **后端**（`src-tauri/src/`）：Rust crate，`main.rs` 仅作为 bin 入口，逻辑全部放在 `lib.rs` 中的 `run()` 函数。crate name 为 `tauri2app_lib`（带 `_lib` 后缀是 Windows 上 cdylib 与 bin 同名冲突的规避，见 Cargo.toml 注释，**不要重命名**）。
- **通信**：前端通过 `@tauri-apps/api` 的 `invoke("命令名", { 参数 })` 调用 Rust；Rust 端用 `#[tauri::command]` 标注函数，并在 `tauri::Builder` 的 `invoke_handler!` 宏中注册。新增命令时两处都要改。

### 权限模型（Tauri 2 的 capabilities）

`src-tauri/capabilities/default.json` 声明窗口可使用的能力。当前应用了 `core:default` 与 `opener:default`（来自 `tauri-plugin-opener`）。**新增插件或调用受限 API 时，必须在该文件追加对应权限**，否则前端调用会被运行时拒绝。

`src-tauri/gen/` 由 Tauri 自动生成（schema 等），不应手工改动；其中 `gen/schemas` 已 gitignore，但 `gen/` 下的其他生成文件可能会被提交。

### 配置耦合点

- 端口 `1420` 在 `vite.config.ts`（`server.port`、`strictPort: true`）和 `tauri.conf.json`（`build.devUrl`）两处必须保持一致。
- `tauri.conf.json` 的 `frontendDist: "../dist"` 指向 Vite 构建产物，路径相对于 `src-tauri/`。
- `tauri.conf.json` 中 `identifier`（`com.admin.tauri2app`）用于打包，发布前应改成正式域名反写。

## TypeScript

启用了 `strict`、`noUnusedLocals`、`noUnusedParameters`，`build` 脚本会通过 `vue-tsc --noEmit` 阻断未使用变量/参数的提交。Vue 文件类型检查依赖 `vue-tsc`，不要换成普通 `tsc`。
