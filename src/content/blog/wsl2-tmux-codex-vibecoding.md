---
title: 'WSL2 重启后，我不想再重搭一遍 Codex 工作台了'
description: '记录我在 WSL2 里用 tmux-resurrect 和 tmux-continuum 保存 Codex vibecoding 工作台的过程，包括安装、配置、使用方式和边界。'
publishDate: 2026-04-08
tags:
  - wsl2
  - tmux
  - codex
  - workflow
language: 'zh-CN'
---

# WSL2 重启后，我不想再重搭一遍 Codex 工作台了

这段时间我在 `WSL2` 里跑 `Codex` 做 `vibecoding`，工作方式其实很固定：

- 一个 pane 跑 `Codex`
- 一个 pane 看项目目录和 Git 状态
- 一个 pane 临时跑命令、看日志或者执行测试

麻烦的地方不在“命令有多难敲”，而在于 Windows 每次重启之后，这套工作台都得重新搭一遍。

重新打开窗口，重新切到项目目录，重新分 pane，重新 attach 或启动会话。事情都不大，但每次开机都先来一轮，久了就很烦。

后来我没有继续折腾更重的方案，而是换了个思路：**别想着把正在执行的任务原样冻住，先把工作台恢复回来。**

> [!note]
> 这篇文章解决的是“重启之后快速回到昨天的工作现场”，不是“让正在执行中的进程无损续跑”。`tmux-resurrect` 和 `tmux-continuum` 更像是保存会话、布局和目录，而不是做进程级快照。

![WSL2 下的 tmux 工作台配图](https://cdn.jsdelivr.net/gh/IT-NuanxinPro/nuanXinProPic@v1.2.52/preview/desktop/%E9%A3%8E%E6%99%AF/%E9%9B%AA%E5%B1%B1/20260406200436.webp)

*有些问题不难，只是重复多了以后会很磨人。*

## 为什么最后选 `tmux-resurrect` + `tmux-continuum`

我最后用的是两件东西：

- [`tmux-resurrect`](https://github.com/tmux-plugins/tmux-resurrect)：负责手动保存和恢复 `tmux` 会话
- [`tmux-continuum`](https://github.com/tmux-plugins/tmux-continuum)：负责定时自动保存，并且在 `tmux server` 启动时自动恢复

这个组合对我来说刚好够用。

我真正想要的，不是“电脑重启后所有任务继续无缝执行”，而是下面这些东西能回来：

- session、window、pane 的结构
- 每个 pane 的工作目录
- 大致固定下来的工作分区

这样我重新进入 `WSL2` 之后，只要执行一次 `tmux`，昨天那套布局基本就回来了，马上就能继续干活。

> [!important]
> `tmux-continuum` 依赖 `tmux-resurrect`。另外它的自动恢复只会在 **tmux server 首次启动** 时触发，单纯执行 `tmux source-file ~/.tmux.conf` 并不会触发自动恢复。

## 这套方案解决了什么

先把结论说清楚，免得装完之后预期不一致。

它能解决的是：

- 开机后不再手动重建 `tmux` 布局
- 重新进入项目目录、会话和 pane 分区
- 让日常的 `Codex` 工作台尽量保持固定

它解决不了的是：

- 正在执行中的构建、脚本、长任务不中断
- `Codex` 交互进程像游戏存档一样断点续跑
- 所有终端输出都百分之百原样回来

`tmux-resurrect` 的官方说明里提到，它默认只会恢复一份比较保守的程序列表，比如 `vim`、`less`、`top`、`htop` 这类；更多程序需要额外配置。对我来说，这反而是合理的，因为我最在意的是**恢复工作台**，不是强行恢复每一个进程的中间态。

## 我的实际做法

我现在的想法很简单：

1. 平时一直在 `tmux` 里工作。
2. 布局稳定下来之后，先手动保存一次。
3. 后面交给 `tmux-continuum` 每隔一段时间自动保存。
4. Windows 重启之后，我重新打开一个 `WSL2` 终端，执行一次命令进入 `tmux`。
5. 之前的工作台自动恢复，我直接继续干活。

注意，这里我接受一个现实：**重启会把正在执行的任务杀掉。**

但这件事对我的影响，远小于每次都重新整理一遍工作区。对日常开发来说，这个取舍是划算的。

## 配置步骤

下面这套步骤我是在 `WSL2 + Ubuntu` 里走通的。

### 1. 安装 `tmux` 和 `git`

如果你还没装：

```bash
sudo apt update
sudo apt install -y tmux git
```

`TPM` 需要 `git`，`tmux` 本体就不用说了。

### 2. 安装 `TPM`

`TPM` 是 `Tmux Plugin Manager`，后面两个插件都靠它来装。

```bash
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm
```

`TPM` 官方要求把初始化语句放在 `~/.tmux.conf` 的最底部，这点别漏。

### 3. 写入 `~/.tmux.conf`

我现在用的是下面这份比较克制的配置：

```bash
# 基础体验
set -g mouse on
set -g history-limit 100000
set -g base-index 1
setw -g pane-base-index 1
setw -g mode-keys vi

# continuum 依赖状态栏钩子，别关
set -g status on

# 快速重载配置
unbind r
bind r source-file ~/.tmux.conf \; display-message "tmux.conf reloaded"

# 插件
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'
set -g @plugin 'tmux-plugins/tmux-continuum'

# 启动 tmux server 时自动恢复
set -g @continuum-restore 'on'

# 每 15 分钟自动保存一次
set -g @continuum-save-interval '15'

# 如果你确实想把 pane 的滚动内容一起存下来，可以打开这一行
# set -g @resurrect-capture-pane-contents 'on'

# TPM 初始化，必须放在最底部
run '~/.tmux/plugins/tpm/tpm'
```

这里有几个点值得单独说一下：

- `set -g status on` 不是摆设。`tmux-continuum` 的自动保存依赖状态栏里的 `status-right` 钩子，状态栏关掉之后，自动保存可能直接失效。
- `tmux-continuum` 最好放在插件列表最后。官方 README 里明确提到，如果别的主题或插件覆盖了 `status-right`，自动保存会停掉，把它放最后更稳。
- `@continuum-save-interval` 默认就是 `15` 分钟，我这里显式写出来，是为了以后回头看配置时不用猜。
- `@resurrect-capture-pane-contents` 我默认没开。因为我主要想恢复布局和目录，不太在意把 pane 的历史输出也带回来。

> [!tip]
> 如果你后面加了主题插件，而且发现自动保存不生效，先别怀疑人生，先检查两件事：状态栏是不是关了，`tmux-continuum` 是不是还在插件列表最后。

### 4. 进入 `tmux`，安装插件

先启动一次 `tmux`：

```bash
tmux
```

默认前缀键是 `Ctrl-b`。进入 `tmux` 之后，执行：

- `prefix + I`：安装插件

也就是先按一次 `Ctrl-b`，再按大写的 `I`。

如果你是改完配置之后想重新加载，也可以在 `tmux` 外执行：

```bash
tmux source-file ~/.tmux.conf
```

或者直接在 `tmux` 里按我上面配好的：

- `prefix + r`：重载配置

### 5. 先手动保存一次

插件装好之后，先把你的工作台整理成你想要的样子，比如：

- 左边主 pane 跑 `Codex`
- 右上角放项目目录
- 右下角放测试或日志

然后执行：

- `prefix + Ctrl-s`：手动保存

之后如果你想验证恢复是否正常，可以直接试一次：

- `prefix + Ctrl-r`：手动恢复

这两个快捷键都是 `tmux-resurrect` 的默认绑定。

### 6. 重启之后怎么恢复

真正到了第二天或者重启机器之后，我的做法只有一条：

```bash
tmux attach 2>/dev/null || tmux
```

这条命令的逻辑很适合日常用：

- 如果已经有 `tmux server` 在跑，就直接 attach
- 如果还没有，就启动一个新的 `tmux server`

而 `tmux-continuum` 的自动恢复，就是在这个“新 server 启动”的时机触发的。

我后来干脆在 `~/.bashrc` 里加了个别名：

```bash
alias ta='tmux attach 2>/dev/null || tmux'
```

这样以后进入 `WSL2`，我只要敲一个：

```bash
ta
```

## 为什么这比“重启后重新开一堆窗口”舒服很多

我之前最烦的，不是某一条命令，而是重新进入状态前那几分钟的碎操作：

- 打开终端
- 进入项目目录
- 重建 pane
- 找回昨天的工作分区
- 想起来哪个窗口该放什么

这些动作单看都不难，但它们会打断进入工作状态的节奏。

换成 `tmux` 之后，至少“工作台形状”稳定了。哪怕任务已经被系统重启打断，我也不用再把桌面重新摆一遍。

对于 `Codex vibecoding` 这种经常一边跑命令、一边切 pane、一边看文件的工作方式，这种稳定感其实很重要。

## 这套方案的边界，我是怎么接受的

说到底，这不是完美方案。

它最大的边界就是：**正在执行的任务还是会死。**

比如下面这些情况，你都不要期待它帮你原地续上：

- 跑到一半的构建
- 正在执行的测试
- 还没结束的脚本
- 交互中的 `Codex` 会话

但我现在已经接受这个边界了，因为我真正要的是：

- 少做重复劳动
- 快速回到熟悉的工作布局
- 不在每天开工前先浪费几分钟整理桌面

如果你的诉求和我一样，那 `tmux-resurrect + tmux-continuum` 已经够用了。

> [!warning]
> 如果你追求的是“系统重启后，长任务继续跑、交互状态继续在、输出不丢”，那这套方案不是答案。它更像是把桌面和工位给你摆回去，不是把昨天那一刻的进程内存冻结下来。

## 最后

我现在在 `WSL2` 里做 `Codex vibecoding`，已经基本固定成一个习惯了：

进入 `WSL2`，敲 `ta`，工作台回来，继续写。

这不是那种很炫的折腾，但它确实把一个每天都会遇到的小烦恼处理掉了。

而且这种改动的好处很直接：它不会改变你的开发方式，只是把你已经习惯的那套工作区，尽量稳稳地留住。

## 参考

- [TPM](https://github.com/tmux-plugins/tpm)
- [tmux-resurrect](https://github.com/tmux-plugins/tmux-resurrect)
- [tmux-continuum](https://github.com/tmux-plugins/tmux-continuum)
