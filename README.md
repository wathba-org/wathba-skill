# wathba — agent skill for the Wathba CLI

<div dir="rtl">

**وثبة** — مهارة وكلاء الذكاء الاصطناعي لواجهة أوامر منصة وثبة. تعلّم أي وكيل (Claude Code، Codex، Cursor، ...) تثبيت الـ CLI واستخدامه، وتفهم الطلبات بالعربية أو الإنجليزية وتنجزها حتى النهاية.

</div>

This skill teaches any AI agent (Claude Code, Codex, Cursor, ...) how to install and operate the [Wathba](https://wathba.info) CLI: authenticate, create projects, activate services (OTP messaging, payments, shipping), integrate capabilities into a member app, manage API keys, and verify results. It understands prompts in **Arabic or English** and replies in the user's language.

## Install

### Recommended (non-technical users): the official installer

The Wathba installer ships this skill with the CLI, signature-verified and version-locked to the binary:

```sh
curl -fsSL https://install.wathba.info/install.sh | bash
```

It installs the `wathba` binary plus this skill into `~/.agents/skills/wathba` and `~/.claude/skills/wathba`. Every `wathba self-update` refreshes the skill so its docs always match the installed CLI.

### Developers: skills CLI

```sh
npx skills add wathba-org/wathba-skill
```

### Developers: Claude Code plugin marketplace

```text
/plugin marketplace add wathba-org/claude-plugins
/plugin install wathba
```

## Contents

- `SKILL.md` — the skill: installation, JSON output contract, typed outcomes, exit codes, core workflows, Arabic/English handling.
- `references/commands.md` — full command surface.
- `references/workflows.md` — end-to-end sequences and the outcome state machine.
- `references/arabic-glossary.md` — Arabic↔English phrase→command mapping.

## Source of truth

This repository is a **read-only mirror**, synced automatically from [`wathba-org/wathba-cli`](https://github.com/wathba-org/wathba-cli) (`internal/skillbundle/agent/wathba`) on every release tag. Do not open content PRs here — change the skill in the CLI repository so it stays version-locked to the binary.
