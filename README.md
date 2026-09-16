<div align="center">

# TextCaddy 文匣

**常用文字，随手即取。**

A keyboard-first local text snippet manager for Windows.

[![License: MIT](https://img.shields.io/badge/License-MIT-blue.svg)](LICENSE)
![Platform](https://img.shields.io/badge/platform-Windows-0078D4)
![Status](https://img.shields.io/badge/status-early%20design-orange)

[中文](#中文) · [English](#english)

</div>

> [!IMPORTANT]
> TextCaddy 尚在早期设计阶段，目前没有可下载的版本。README 中描述的功能均为规划目标。
>
> TextCaddy is in the early design stage. No downloadable build is available yet. Features described here are planned goals.

## 中文

### TextCaddy 是什么？

TextCaddy（文匣）是一款面向 Windows 的轻量级本地文本库。它帮你保存地址、邮箱、常用话术、命令和代码片段等重复使用的文本，并通过全局快捷键在当前工作流中快速调用。

TextCaddy 不会监听或记录剪贴板历史。你保存的内容属于独立文本库，不会因新的复制操作而丢失。

### 为什么做 TextCaddy？

日常使用电脑时，我们经常需要反复输入同一批文本。普通剪贴板只适合短暂中转，内容很容易被下一次复制覆盖；剪贴板历史工具又往往会收集过多不必要的信息。

TextCaddy 只保存你主动加入的内容，并专注于一件事：用尽可能少的键盘操作找到并复制常用文本。

### 计划中的核心体验

1. 使用全局快捷键在鼠标光标附近打开悬浮面板。
2. 浏览当前分组，或直接输入关键词搜索标题和内容。
3. 按数字键选中、多选或取消选中当前页中的条目。
4. 按 `Ctrl+C` 将选中内容按顺序合并复制，然后自动收起面板并返回原应用。
5. 在原应用中按 `Ctrl+V` 完成粘贴。

`Ctrl+Shift+C` 将复制内容但保持面板打开，`Esc` 则关闭面板且不改变剪贴板。具体的全局唤起快捷键将在实现阶段确定，并支持自定义。

### 计划功能

- 靠近光标显示的键盘优先悬浮面板
- 使用数字键快速选择和组合多条文本
- 分组、搜索、排序和分页
- 独立的托盘管理窗口
- 可配置的多条文本合并分隔符
- 敏感条目遮挡和临时查看
- 敏感内容复制后定时清空剪贴板
- 全部数据仅保存在本机

### 产品边界

TextCaddy 不是剪贴板历史工具，也不是专业密码管理器。

- 不自动记录用户复制过的内容
- 不提供浏览器自动填充或账号登录管理
- 首版不提供云同步、团队共享或跨平台支持
- 首版不保存富文本、图片或文件
- 不接管其他应用中的普通 `Ctrl+V` 行为

> [!WARNING]
> 即使 TextCaddy 提供基础隐私保护，也不应代替专业密码管理器。请不要用它保存重要账号的唯一密码副本。

### 路线图

- [x] 明确产品定位与 MVP 边界
- [ ] 完成产品需求与交互规格
- [ ] 确定 Windows 技术栈和应用架构
- [ ] 实现全局快捷键、悬浮面板和焦点恢复
- [ ] 实现文本库、分组、搜索和数字键多选
- [ ] 实现本地数据保护和剪贴板定时清理
- [ ] 提供首个可测试的 Windows 版本

### 参与项目

TextCaddy 正处于早期设计阶段。欢迎通过 [Issues](https://github.com/yxxxxx0123/textcaddy/issues) 提交使用场景、交互建议和功能讨论。在开始大型功能开发前，请先创建 Issue 说明问题和解决思路，避免重复工作。

---

## English

### What is TextCaddy?

TextCaddy is a lightweight, local text snippet library for Windows. It keeps frequently used addresses, email addresses, canned responses, commands, code snippets, and other reusable text within quick reach through a global shortcut.

TextCaddy does not monitor or collect clipboard history. Its library contains only the text you intentionally save, so a new copy operation never overwrites your stored snippets.

### Why TextCaddy?

Many everyday computer tasks involve typing the same pieces of text again and again. A regular clipboard is only a temporary handoff and is overwritten by the next copy operation. Clipboard history tools, meanwhile, often collect far more information than needed.

TextCaddy stores only what you explicitly add and focuses on one job: finding and copying reusable text with as few keyboard actions as possible.

### Planned core workflow

1. Press a global shortcut to open a floating panel near the mouse pointer.
2. Browse the active group or start typing to search snippet titles and content.
3. Use number keys to select, multi-select, or deselect entries on the current page.
4. Press `Ctrl+C` to combine the selected entries in order, copy the result, close the panel, and return focus to the previous application.
5. Press `Ctrl+V` in that application to paste normally.

`Ctrl+Shift+C` will copy while keeping the panel open. `Esc` will close the panel without changing the clipboard. The default global shortcut will be chosen during implementation and will be customizable.

### Planned features

- A keyboard-first floating panel positioned near the pointer
- Fast number-key selection and combination of multiple snippets
- Groups, search, ordering, and pagination
- A separate system-tray management window
- Configurable separators when combining snippets
- Masking and temporary reveal for sensitive entries
- Timed clipboard clearing after copying sensitive content
- Local-only data storage

### Non-goals

TextCaddy is neither a clipboard history tool nor a dedicated password manager.

- It will not automatically record copied content.
- It will not provide browser autofill or login management.
- The first release will not include cloud sync, team sharing, or cross-platform support.
- The first release will not store rich text, images, or files.
- It will not intercept ordinary `Ctrl+V` behavior in other applications.

> [!WARNING]
> TextCaddy may provide basic privacy safeguards, but it is not a replacement for a dedicated password manager. Do not use it as the only place where important account passwords are stored.

### Roadmap

- [x] Define the product direction and MVP boundaries
- [ ] Complete the product requirements and interaction specification
- [ ] Select the Windows technology stack and application architecture
- [ ] Implement the global shortcut, floating panel, and focus restoration
- [ ] Implement the snippet library, groups, search, and number-key multi-selection
- [ ] Implement local data safeguards and timed clipboard clearing
- [ ] Publish the first testable Windows build

### Contributing

TextCaddy is currently in its early design stage. Use [Issues](https://github.com/yxxxxx0123/textcaddy/issues) to share workflows, interaction ideas, and feature proposals. Before starting a substantial feature, please open an issue describing the problem and the proposed solution so that we can discuss it and avoid duplicated work.

## License

TextCaddy is released under the [MIT License](LICENSE).
