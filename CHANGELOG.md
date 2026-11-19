# Changelog · 更新日志

Public releases of Palmhelm for Mac. Installers are on [Releases](https://github.com/forsakesoul/palmhelm/releases).

Palmhelm for Mac 的公开版本。安装包见 [Releases](https://github.com/forsakesoul/palmhelm/releases)。

[English](#english) · [中文](#中文)

# English

## 1.0 (153) · 2026-11-19

Built-in Palmhelm Bridge 0.3.13 (previously 0.3.12).

### New
- Tells you when a new version is out: a dismissible banner at the top of the main window lets you download it, read what changed, or skip that version. In Settings › About you can turn off automatic checks or check now. It only asks GitHub for the latest version number, sends no personal data, and never downloads or installs anything by itself.

### Improved
- When you first open a conversation whose latest entries are taken up by long tool output, Palmhelm now loads a few more pages on its own and shows the full conversation, instead of waiting for you to click "Load earlier".
- The note after a mid-turn message is delivered now reads plainly: "Delivered to the working terminal; not yet confirmed in the current turn".

### Fixed
- The English interface used plurals for a count of 1 (such as "1 rules" and "1 read errors"); 49 strings corrected.
- New messages sometimes did not appear after earlier pages were loaded automatically.

## 1.0 (149) · 2026-11-18

Built-in Palmhelm Bridge 0.3.12 (previously 0.3.9).

### New
- Approval options, Issue task templates, history and inbox text follow the app language (Chinese / English).
- You can message any agent directly or dispatch an Issue task to it; executor and advisor panes are no longer refused.

### Improved
- Conversations, first sync, the terminal and the sidebar show placeholder skeletons while loading instead of a blank area.
- The message ledger and the conversation show the same current status; opening or returning to the ledger window refreshes it and fills in delivery receipts.

### Fixed
- A loaded conversation was covered by the skeleton after the window lost focus.
- Finished "result unknown" or paused commands stayed pinned to the end of the conversation.
- The built-in Bridge's child processes did not exit when the app quit or restarted.

## 1.0 (141) · 2026-11-17

Built-in Palmhelm Bridge 0.3.9 (previously 0.3.4).

### New
- Choose the app language (Chinese / English) in Settings; it applies immediately and follows the system language by default.
- Message ledger: see queued, delivered and stopped messages per target, with Markdown bodies, a copy button on each entry, and paging for long entries.
- Dispatch a GitHub issue or a hand-written task to an agent from the Mac.
- Manage several Codex / Claude account sources in Settings; usage is summed per account.
- Send mode: tap the hint line above the input to choose "Send after this turn", "Insert now", "Within 1 hour", "Within 24 hours" and more. The current mode is checked, and unavailable modes say why.

### Improved
- The conversation header collapses and expands with the scroll direction; the title shows the agent's identity and the model capsule drops its source prefix.
- Long replies collapse and expand with an icon; when expanded, the time and the collapse button share one line.
- Queued messages show a status card (queue position, paused, stopped and so on), including messages queued from other devices.
- Space groups in the sidebar are ordered by first run, pinning and drag-and-drop.
- Conversation titles skip generic or unsafe terminal titles.
- Long conversations open faster, and unchanged transcripts are no longer sent again.
- Scrolling a conversation and refreshing targets in the background no longer recompute the whole page, so long conversations stay smooth.

### Fixed
- The same message appeared as two bubbles in long conversations.
- An agent back to idle kept its target occupied because of an earlier "result unknown", so later messages could not be sent. The occupation is now released automatically with a prompt to check the terminal; nothing is resent.
- Sending while an input method was composing sent early or dropped characters.
- The "Back to latest" button was hidden behind the input field.
- Approval and reminder expiry times were inaccurate.
- The background Bridge's runtime identity record could be lost.
- Messages too large to display now say so.

## 1.0 (107) · 2026-11-12

First public release, notarized by Apple. Built-in Palmhelm Bridge 0.3.4.

# 中文

## 1.0 (153) · 2026-11-19

内置 Palmhelm Bridge 0.3.13（上一版为 0.3.12）。

### 新增
- 有新版本时提醒你：在主窗口顶部显示可关闭的提示，可以下载新版、查看更新内容或跳过此版本；设置 › 关于里可以关闭自动检查，也可以立即检查。只向 GitHub 查询版本号，不含任何个人数据，不会自动下载或安装。

### 改进
- 第一次打开会话时，如果最近几条被大段工具输出占满，会自动往前多取几页，直接显示完整的对话，不用再手动点“加载更早的记录”。
- 插话送达后的说明改成更直白的“已送达工作中终端，尚未确认并入本轮”。

### 修复
- 英文界面在数量为 1 时仍用复数（如“1 rules”“1 read errors”），共修正 49 处。
- 自动补页后新到的消息有时不显示。

## 1.0 (149) · 2026-11-18

内置 Palmhelm Bridge 0.3.12（上一版为 0.3.9）。

### 新增
- 审批选项、Issue 任务模板、历史记录与收件箱正文按 App 语言（中文 / English）显示。
- 可以直接给任何 agent 发消息或派 Issue 任务，不再拒绝执行者、顾问类窗格。

### 改进
- 打开会话、首次同步、终端与侧栏加载时显示占位骨架，不再空白。
- 消息账本与会话显示同一当前状态；打开或切回账本窗口时自动刷新并补上送达回执。

### 修复
- 窗口失去焦点后，已加载的会话被骨架遮住。
- 已结束的“结果未知”或已暂停的命令一直停在会话末尾。
- 退出或重启 App 时，内置 Bridge 的子进程没有一起结束。

## 1.0 (141) · 2026-11-17

内置 Palmhelm Bridge 0.3.9（上一版为 0.3.4）。

### 新增
- 设置里可以选 App 语言（中文 / English），切换立即生效；默认跟随系统语言。
- 消息账本：按目标查看排队、送达、停止的记录，正文按 Markdown 显示，每条可单独复制，长条目可分页阅读。
- 从 Mac 把 GitHub issue 或手写任务派给 agent。
- 设置里可管理多个 Codex / Claude 账号来源，用量按账号汇总。
- 发送方式：点输入框上方的提示行即可选择“本轮结束后发送 / 立即插入 / 1 小时内 / 24 小时内”等，当前方式带勾，不可用的写明原因。

### 改进
- 对话头部随滚动方向收起与展开；标题显示 agent 身份，模型胶囊去掉来源前缀。
- 长回复折叠与展开改用图标，展开后时间与收起按钮同一行。
- 排队消息显示状态卡（排队位置、暂停、已停止等），来自其他设备的排队消息也会显示。
- 侧栏空间分组按首次运行、置顶与拖动排序。
- 会话标题跳过通用或不安全的终端标题。
- 长会话首屏加载更快，未变化的会话记录不再重复传输。
- 滑动对话、后台刷新目标时不再整页重算，长对话更流畅。

### 修复
- 长会话里同一条消息出现两个气泡。
- agent 已回到空闲，却因上一条“结果未知”一直占用目标、后续消息发不出去：现在自动解除占用，并提示去终端核对，不会重发。
- 输入法组字时发送或丢字。
- “回到最新”按钮被输入框挡住。
- 审批与提醒到期时间不准。
- 后台 Bridge 运行时身份记录可能丢失。
- 消息过大无法显示时给出说明。

## 1.0 (107) · 2026-11-12

首个公开版本，经 Apple 公证。内置 Palmhelm Bridge 0.3.4。
