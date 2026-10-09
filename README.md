<h1 align="center">gray-plannotator</h1>
<p align="center">Annotated plan review — the user marks up the agent's plan before implementation.</p>
<p align="center">
  <a href="https://github.com/vstaln/gray-plannotator/blob/main/LICENSE"><img alt="MIT License" src="https://img.shields.io/badge/license-MIT-blue.svg"></a>
  <img alt="gray plugin" src="https://img.shields.io/badge/gray-plugin-7aa2f7.svg">
  <img alt="rust" src="https://img.shields.io/badge/built%20with-rust-orange.svg">
</p>

Terminal port of [`@plannotator/pi-extension`](https://github.com/backnotprop/plannotator):
the interactive plan-review loop, without the GUI. The agent writes its
plan to `.gray/plans/<ts>.md`, you annotate it in `$EDITOR`, and the plugin
diffs your markup back to the agent as a revision prompt until you accept.

## The loop

```text
/plan-review     start: the agent is asked to write .gray/plans/plan-<ts>.md
                 (then re-run it once the file appears — it's adopted as the baseline)
$EDITOR          annotate the file: `<!-- note -->` comments and/or direct edits
/plan-review     submit: your comments + text edits become a revision prompt;
                 the agent rewrites the plan
/plan-review     repeat per round — "revision N is in" when the file changes
/plan-accept     approve: the agent is told to implement the plan
```

Commands:

- `/plan-review` — bare call advances the loop one step (start → adopt →
  submit annotations → next round). Subcommands: `status`, `path`, `cancel`.
- `/plan-accept` — approves the plan at its current contents (final edits
  included) and clears the review; the agent is prompted to implement.

While a review is open, the `prompt/context` hook injects a short
"do not implement yet — awaiting annotations" note every turn, so the
agent can't run ahead of the reviewer.

## Install

```sh
cargo build --release
# symlink or copy the binary where gray loads plugins, or point
# `gray plugin check` / your gray.yml at this directory:
gray plugin check target/release/gray-plannotator
```

No setup, no credentials, no config.

## How it works

- Sidecar speaks gray wire v1 (`protocol: "1.1"`): `plugin/manifest`,
  `prompt/context`, `command/run`, `plugin/shutdown` as JSON lines on stdio.
- `/plan-review` returns `{"prompt": …}` results — the host submits them as
  a user turn, which is how annotations reach the agent as a revision
  request (and how `/plan-accept` starts implementation).
- State lives with the plans at `<cwd>/.gray/plans/.review.json`; accept or
  `cancel` removes it. Plan files are kept as an archive.
- Edit attribution is by phase: a file change inside `awaiting_annotations`
  is treated as user markup; inside `awaiting_revision` it's the agent's
  rewrite. The context injection tells the agent not to touch the file
  while the user owns it.
- Feedback extraction: new `<!-- … -->` comments (with the nearest plan
  line for context) plus a multiset line diff of the comment-free text, so
  reordered paragraphs don't read as edits.

## Works alongside gray-plan

`gray-plan` is a read-only tool gate; this plugin is a review workflow —
separate concerns, composable. Note the interaction: under plan mode the
agent's `write` of the plan file is denied, so every agent-facing prompt
carries a fallback ("reply with the plan text; the user saves it to the
path").

## Limits

- The plugin can't open `$EDITOR` itself (a `command/run` answers inside
  ~30s and the TUI owns the terminal) — it tells you the path; you edit in
  your own editor and re-run `/plan-review`.
- If the agent edits the plan file while the user is annotating, the diff
  attributes that change to the user (phases attribute authorship; there
  is no file watching).
- Annotation round-trips are user-driven: nothing happens between
  `/plan-review` calls.
