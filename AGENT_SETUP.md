# How the agent is set up

Before you start generating code, it helps to know where your agent's
instructions come from and how to check what it's actually been told. 

## The instruction files

| File | Who writes it | What it does |
|---|---|---|
| `AGENTS.md` | Your instructor | The agent's rules for this project: check the spec first, follow the Nextflow conventions, log every decision, don't touch your review fields |
| `CLAUDE.md` | Your instructor | Contains one line, `@AGENTS.md`, so Claude Code loads `AGENTS.md` |
| `specifications.md` | You | What to build. The agent treats it as the context for how to build the pipeline |
| `review/decisions.md` | Both | The agent fills in the agent fields; you fill in the fields at the bottom |

### How Claude Code reads them

- **`CLAUDE.md` is loaded automatically** at the start of every session started
  in this directory. You don't need to paste it in or tell the agent to read it.
- **`@path` imports another file.** `@AGENTS.md` in `CLAUDE.md` pulls in the full
  contents of `AGENTS.md`. The rules live in `AGENTS.md` so that every agent
  (not just Claude) reads the same ones.
- **Instructions can also come from outside this repo.** Claude Code also loads
  `~/.claude/CLAUDE.md` (your personal instructions, applied to every project)
  and `CLAUDE.local.md` in the project directory (personal and not meant to be
  committed). If you've set either of these up for other work, they apply here
  too.
- **Claude Code may keep its own memory.** It can save notes between sessions
  (for example, something you corrected it on last week). Those notes are loaded
  in later sessions too.

### Checking what's loaded

Run `/memory` inside Claude Code. It lists every instruction and memory file
loaded in the current session and lets you open any of them. Do this once at the
start, and again if the agent does something that seems to ignore `AGENTS.md`.

## Rules for you

- **Don't edit `AGENTS.md` or `CLAUDE.md`.** They're part of the assignment. If
  something in them is blocking you, ask your instructor.
- **Don't run `/init`.** It generates a `CLAUDE.md` by scanning the repo, which
  would replace or bury the course instructions.
- **Don't let personal instructions contradict the course ones.** If your
  `~/.claude/CLAUDE.md` says something like "skip comments" or "don't ask, just
  do it," the agent gets conflicting rules and may stop logging decisions. Use
  `/memory` to check.
- **Put project conventions in the spec.** If you want the agent to do
  something a particular way (naming, a container registry, a cluster profile),
  write it in the **Implementation Conventions** section of
  `specifications.md`. That's the only place the agent is told to treat as
  overriding `AGENTS.md`, and it keeps the decision visible in your spec rather
  than hidden in a chat.

## Instructions aren't guarantees

`AGENTS.md` tells the agent what to do; it doesn't force it to. Agents
sometimes skip a decision, log a vague one ("used standard defaults"), or forget
a rule late in a long session. If that happens:

- Point it back at the rule: *"AGENTS.md section 3 says to log every flag you
  chose. Which ones did you leave out of decisions.md?"*
- Start a fresh session (`/clear`) if the conversation has gotten long. The
  instruction files are reloaded; whatever was said in the chat is not.

## Working with permissions

By default, Claude Code asks before it edits a file or runs a command. Read
those prompts rather than approving them on reflex. They're your first look at
what the agent is about to change.

- **Shift+Tab** cycles through modes. *Plan mode* has the agent describe what it
  intends to do without changing anything, which is useful before a large
  regeneration. *Accept edits* stops asking about file edits (it still asks
  about commands).
- **Never approve a full (non-stub) `nextflow run`** unless you meant to start
  one. `AGENTS.md` tells the agent not to run one unasked, but the permission
  prompt is your last check.

## Other agents

| Agent | Reads |
|---|---|
| Claude Code | `CLAUDE.md` (which imports `AGENTS.md`) |
| Codex, Cursor, GitHub Copilot, and most others | `AGENTS.md` directly |

The details (how to view loaded instructions, permission modes, personal
instruction files) differ by tool, so look up the equivalent of each section
above in your agent's documentation. The rules for you are the same.
