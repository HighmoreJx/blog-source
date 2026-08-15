---
title: TUI 便捷的终端管理
date: 2026-08-15 16:37:03
tags:
  - TUI
  - macOS
  - Prowl
  - 效率工具
categories:
  - 工具
---

同时开着好几个项目、每个项目又跑着一两个 AI coding agent 的时候，普通终端的一堆 tab 很快就乱成一团：哪个 tab 是哪个项目、agent 跑到哪一步、要重跑测试得先翻半天。最近换用 [Prowl](https://prowl.onev.cat/) 管理终端，配合自己做的一个 Finder 右键快捷动作，工作流顺畅了很多，记一下。

## Prowl 是什么

Prowl 是一个原生 macOS 终端应用，专门为「和 AI coding agent 协作」这件事打磨。作者是 [@onevcat](https://github.com/onevcat)，基于 Supacode fork 而来，免费且开源。标语很传神：**Quiet paws. Sharp claws.**（安静的爪子，锋利的爪子）——界面轻量不打扰，能力却不弱。

它解决的核心问题：**给每个项目一个独立、有序的工作区**，而不是把所有东西塞进通用终端 tab 里互相打架。

几个用起来最舒服的点：

- **多项目工作区**：一屏管理多个项目，各自独立，互不干扰
- **Canvas Shelf**：置顶/收藏常用项目的面板
- **每项目专属动作按钮**：Run / Test / Build / Release / Open App / Diff，按项目定制，一键触发
- **coding agent 集成**：直接在终端里跑 `codex` 之类的 agent，改文件、重试逻辑都在同一个界面
- **实时输出 + Diff 视图**：测试/构建结果内联显示，改动前后一目了然

安装很简单，直接下 DMG，或 Homebrew：

```bash
brew install --cask onevcat/tap/prowl
```

## 痛点：从 Finder 到终端那一步

用顺 Prowl 之后，剩下一个小摩擦：在 Finder 里翻到一个项目文件夹，想「用 Prowl 打开它」，还得手动切到 Prowl、再找路径添加。多做几次就烦。

于是做了个 macOS Finder 快捷动作（Quick Action）：**选中文件夹 → 右键 → 添加到 Prowl**，Prowl 自动启动或切到前台，把选中的文件夹依次打开成终端。

仓库在这：**[github.com/HighmoreJx/prowl-finder-quick-action](https://github.com/HighmoreJx/prowl-finder-quick-action)**

### 它怎么工作

核心就一个 shell 脚本，调用 Prowl 自带的 CLI 打开目录：

```bash
for candidate in \
  "/Applications/Prowl.app/Contents/Resources/prowl-cli/prowl" \
  "/usr/local/bin/prowl"
do
  if [ -x "$candidate" ]; then cli="$candidate"; break; fi
done

for selected_path in "$@"; do
  "$cli" open "$selected_path"
done
```

优先用 Prowl.app 内置的 CLI，找不到再退回 `/usr/local/bin/prowl`（后者可在 Prowl 的 **Settings → Advanced → Install Command Line Tool** 装）。

几个细节处理得比较稳：

- **多选**：一次选多个文件夹全部处理，最后一个成功打开的保持聚焦
- **去重**：Prowl 已管理的目录不重复加入；选的是现有项目的子目录时，会在该项目里为子目录开终端
- **中文/空格路径**：都能正确处理
- **失败继续**：某个路径失败不影响其余，最后弹一次汇总告警
- **CLI 缺失兜底**：找不到 Prowl CLI 会弹窗提示，而不是静默失败

## 在别的 Mac 上用

给其他 Mac 用很简单：

1. 在另一台 Mac 安装 Prowl，放在 `/Applications/Prowl.app`
2. 从仓库拿到 `添加到 Prowl.workflow`（clone 仓库，或直接下载）
3. 双击 `.workflow` 安装；也可以复制到：
   ```text
   ~/Library/Services/
   ```
4. 前往 **系统设置 → 键盘 → 键盘快捷键 → 服务**，启用「添加到 Prowl」
5. 设一个不冲突的快捷键，例如 `⌃⌥⌘P`

之后在 Finder 里选中文件夹，右键或按快捷键即可。

## 小结

Prowl 把「多项目 + 多 agent」的终端管理理顺了，Finder 快捷动作又把「从文件夹进 Prowl」这最后一步磨平。两个加起来，日常在项目间跳转基本不用再手动敲路径了。工具很小，但每天都在省事。
