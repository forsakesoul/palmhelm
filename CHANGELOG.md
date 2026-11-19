# Changelog

Public releases of Palmhelm for Mac. Installers are on [Releases](https://github.com/forsakesoul/palmhelm/releases).

[中文版](CHANGELOG.zh-CN.md)

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
