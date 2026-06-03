# 发布信息

## 标题

Windows 上 Codex Desktop 的 Chrome 和 Computer Use 插件不可用：完整排查与修复

## 摘要

Codex Desktop 更新后，Chrome 和 Computer Use 插件可能同时显示不可用。本文记录一次完整排查：从 `config.toml`、bundled plugin marketplace、`.agents\plugins\marketplace.json`、`latest` 指针，到 `codex://` 协议注册表修复，最终让 `browser`、`chrome`、`computer-use` 恢复为 `installed, enabled`。

## 标签

Codex, Windows, Chrome 插件, Computer Use, OpenAI, PowerShell, 插件修复, Native Messaging

## 分类建议

开发工具 / Windows / AI 编程助手

## CSDN 专栏建议

AI 编程工具

## Git 仓库文件名建议

`codex-windows-computer-use-chrome-plugin-fix.md`

## README 链接文案

- [Windows 上 Codex Desktop 的 Chrome 和 Computer Use 插件不可用：完整排查与修复](./codex-windows-computer-use-chrome-plugin-fix.md)

## 发布前检查

- 文章正文：`outputs/codex-windows-computer-use-chrome-plugin-fix.md`
- 代码块使用 PowerShell / TOML / text 标注。
- 没有暴露用户私密 token、cookie、密钥。
- 示例路径已尽量使用 `%USERPROFILE%` 或 `$env:USERPROFILE`。
- 个别版本号保留为实际案例值，并提醒读者按自己机器路径调整。
