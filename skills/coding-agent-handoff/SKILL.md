---
name: coding-agent-handoff
description: Writes a handoff of the current coding session with four sections - what changed, what was actually verified, what is blocked or unverified, and what to do next. Use when the user is stepping away, pausing, switching sessions or agents, or asks for a handoff, a wrap-up, or where things stand.
---

# Coding agent handoff

Write a handoff that lets someone continue this work without reading the conversation: the user coming back later, a teammate, or another agent session. The handoff is only useful if the reader can trust that everything under **Verified** really ran.

## Before writing

- You may run read-only commands to list changes, such as `git status` and `git diff --stat`.
- Do not run tests, builds, linters or any other check in order to fill in the handoff. Report the session as it stands.
- Do not create, edit or delete files, commit, or use the network. The handoff is a reply in the conversation. Save it to a file only if the user asks for that, at the path they give.

## Format

Reply with these four sections, in this order, under these headings. Write `None` in a section that has nothing to report. Never leave a section out.

### Changes

One line per file changed in this session: the path relative to the repository root, then what changed and why. Mark changes that were already uncommitted when the session started as `pre-existing`. Say whether the work is committed (give the commit hash) or uncommitted.

### Verified

Only checks that meet all three conditions:

1. You ran the check in this session.
2. It ran after the last change it covers.
3. You read its result: an exit status or a clear pass or fail line.

For each one give the exact command, its scope if it covered only part of the change, and the result. A check that failed and meets the three conditions belongs here too, marked `FAILED`, with the failing line.

### Blocked or unverified

Everything else a reader might assume is done:

- Checks that were not run, were skipped, or ran on a subset that does not cover the change.
- Checks that ran before a later change. Mark them `stale`.
- Checks that are still running, or whose output you did not see in full.
- Results you did not produce yourself. Name the source: `reported by user`, `from previous session`, or `CI at <commit>`. A CI status describes the current work only when that commit is HEAD and nothing is uncommitted. Even then it stays in this section, with its source.
- Conclusions you reached by reading code without running it. Mark them `inferred`.
- Decisions or access you need from the user, one line each, written as a question they can answer.

### Next steps

A numbered list, in the order to do them. Make each step concrete enough to start without the conversation: the command to run or the file to open. Put the unverified checks that are worth running first.

## Rules

- When you are unsure whether something belongs under Verified, put it under Blocked or unverified.
- Do not say or imply outside the Verified section that tests pass, the build works or the bug is fixed.
- No secrets: no tokens, keys, passwords, cookies or signed URLs. Refer to them by name, for example "the API key in `.env`".
- No absolute paths that contain a user name or home directory. Use paths relative to the repository root.
- Keep it short: lines, not paragraphs. Do not summarize the conversation, and do not add an offer to help at the end.

`examples/handoff-example.md` shows a complete handoff.
