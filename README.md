# Plan Agent

A pointer to the **`Plan` sub-agent built into [Claude Code](https://code.claude.com/docs/en/overview)**. There is nothing to install. This repo documents when and how to use it from a hotel-operations workflow.

> **Note:** Unlike the other th1-ai agents, `Plan` is not a custom sub-agent definition. It ships with Claude Code. There is no agent file to copy, and nothing to configure. To use it from a scheduled, unattended hotel agent, see [Run it on its own](#run-it-on-its-own).

---

## What it does

`Plan` is a read-only research and planning sub-agent. Claude Code uses it mainly in **plan mode**: when Claude needs to understand your files before it proposes a change, it hands the research to `Plan`. The research runs in a separate context window, so the main conversation stays short.

What you get back:

- A step-by-step plan for the change
- The files and call sites that the change touches
- The trade-offs, risks and likely failure modes
- No edits. `Plan` cannot use the Write or Edit tools.

The point is to think *before* changing anything, especially where a wrong edit costs real time (live PMS bookings, finance logs, guest-facing email drafts).

Three facts from the docs that matter in practice:

- **Model:** `Plan` uses the same model as your main conversation.
- **No `CLAUDE.md`:** `Plan` starts without your `CLAUDE.md` files, to keep research fast. If a rule in `CLAUDE.md` must shape the plan (for example "never write to the PMS without approval"), say it in your prompt too.
- **No preloaded skills:** built-in sub-agents do not preload skills.

---

## What it is **not**

`Plan` is not for:

- **Searching.** Use [`explore-agent`](https://github.com/th1-ai/explore-agent) for "where is X defined".
- **Writing the change.** `Plan` returns a plan. The main session (or you) executes it after you approve.
- **Trivial one-line fixes.** Just edit the file.
- **Open-ended research.** Use [`general-purpose-agent`](https://github.com/th1-ai/general-purpose-agent) for "should we even do this?".

---

## How to use it

### Option 1: plan mode (recommended)

Plan mode is where Claude Code uses `Plan` by design. Claude reads and explores, then presents a plan and waits. It does not edit your files until you approve.

Enter plan mode in one of these ways:

- Press `Shift+Tab` until the mode indicator shows plan mode.
- Prefix one prompt with `/plan`:

  ```
  /plan add a stale-lead chase to the CRM pipeline without breaking the existing nightly job
  ```

- Start the session in plan mode:

  ```bash
  claude --permission-mode plan
  ```

When the plan is ready, Claude asks how to proceed. You can approve it (Claude then starts the edits), or choose to keep planning and say what to change. Press `Shift+Tab` again to leave plan mode without approving.

### Option 2: name it in a normal session

There is no special syntax. Name the sub-agent and Claude typically delegates:

```
Use the Plan subagent to design how we'd add price-drop alerts to the agent-notification flow. Do not change any files.
```

To make sure `Plan` runs, rather than leaving the choice to Claude, type `@` and pick it from the list, or type `@agent-Plan`.

A prompt such as "think it through before you write any code" does not guarantee that `Plan` runs. Claude can also plan in the main conversation. If you want the guarantee that nothing is edited, use plan mode.

---

## Verify it works

1. Open Claude Code in any project folder.
2. Press `Shift+Tab` until plan mode is on.
3. Ask: `How would you add a new sender rule to the email triage? Plan only.`
4. Check that Claude returns a plan and asks for approval, and that `git status` shows no changed files.

You do not always see `Plan` named in the transcript. Claude delegates to it only when it needs to research your files; for a small question it can answer directly.

---

## Suggested usage with hotel agents

Plan first, before:

- Adding a new skill to an existing agent (e.g. extending `email-triage-agent` to handle a new sender pattern)
- Wiring a new tool (e.g. swapping the PMS adapter from one provider to another)
- Changing an SOP that several agents depend on
- Any change that touches finance logging or live PMS writes
- Any change that spans several files or has a critical rule to keep (no double-charging guests, no change to a live reservation without approval)

Skip it for:

- Editing copy in a draft email
- Adjusting a single threshold or label
- Changes confined to one file with an obvious shape

---

## Run it on its own

`Plan` cannot change anything, so it is the safe first step for a hotel agent that runs on a schedule: the run reads the files, prints a plan, and stops. You read the plan and approve it, or not.

[`AUTONOMY.md`](AUTONOMY.md) sets how far this goes. It ships at `Level: 1` (scheduled, report only). Its Level 3 is empty on purpose: a plan is never executed because it exists.

### The scheduled command

Run this from the folder of the hotel agent that you want a plan for (for example your clone of `email-processor-agent`):

```bash
claude -p "Plan only. Read AUTONOMY.md first and obey it. Review this agent's files and the last entries in runs/LOG.md. Plan the changes that would fix the problems you find. Start the report with a section named 'Needs you'." \
  --permission-mode plan \
  --permission-prompts none \
  --settings .claude/settings.example.json \
  --max-turns 20 \
  --output-format json
```

What each part does, from the Claude Code docs:

- `-p` runs without the interactive screen. It still loads the folder's `CLAUDE.md`, skills, hooks and MCP servers, as an interactive session does.
- `--permission-mode plan` starts in plan mode. The docs state that plan mode keeps its blocks in `-p` runs, so edits stay blocked.
- `--permission-prompts none` tells Claude Code that nobody can answer a prompt. A call that needs approval is denied and the run continues. Needs Claude Code v2.1.259 or later.
- `--settings` loads the permissions file for this run only (see below).
- `--max-turns` stops a run that does not finish. `--max-budget-usd` is a second cap for runs that are billed to an API key.
- `--output-format json` returns the plan in `result`, the denied calls in `permission_denials`, and an estimate in `total_cost_usd`.

Checked on Claude Code v2.1.323 in a 3-file test folder: the run printed the plan, changed no file, and a read of `.env` was denied. It took 8 turns. Your folder and model give different numbers.

Do not add `--bare` to this command. Bare mode skips `CLAUDE.md` and skills, and it does not use your subscription login.

### Least-privilege permissions

[`.claude/settings.example.json`](.claude/settings.example.json) is an example. It has the name `settings.example.json` so that a clone of this repo changes nothing. Copy it into the hotel agent's `.claude/` folder and pass it with `--settings`, as the command above does.

- `permissions.defaultMode: "plan"` starts the run in plan mode.
- `permissions.deny` removes `Edit`, `Write`, `NotebookEdit`, `WebFetch` and `WebSearch`, and denies `curl`, `wget`, `rm`, `git push` and reads of `.env` files.
- `useAutoModeDuringPlan: false` matters for a hotel. When it is on (the default where auto mode is available), a classifier can approve shell commands during planning. With it off, a command outside the built-in read-only set needs approval, and an unattended run denies it. Your PMS or mail CLI then cannot write during a plan run. Claude Code reads this key from `--settings`, user or local settings, not from a project `.claude/settings.json`.

If the plan needs live data, add an allow rule for one read-only command of your own CLI, for example `"allow": ["Bash(your-pms-cli arrivals *)"]`. The docs warn that a Bash rule matches the command text and is not a security boundary. Give the scheduled run read-only credentials where your tools support that.

### A scheduler example

This is an example for cron on macOS or Linux. The Claude Code docs do not describe cron for `claude -p`, and this line was not run under cron here. The two `grep` checks are the kill switch. They were tested against `Level: 1`, `Level: 3`, `Level: 0`, `Level: PAUSED`, an unknown value, two `Level:` lines and a missing file. Only one line with `Level: 1`, `Level: 2` or `Level: 3` starts a run.

```cron
# Mondays 06:00: one plan-only run. Use full paths; cron has a short PATH.
0 6 * * 1 cd /path/to/your-hotel-agent && [ "$(grep -c '^Level:' AUTONOMY.md)" = 1 ] && grep -qE '^Level: [123]$' AUTONOMY.md && /path/to/claude -p "Plan only. Read AUTONOMY.md first and obey it. Start the report with 'Needs you'." --permission-mode plan --permission-prompts none --settings .claude/settings.example.json --max-turns 20 --output-format json > runs/plan-report.json 2>&1
```

Claude Code also has its own schedulers (cloud scheduled tasks, Desktop scheduled tasks and `/loop`). See [Run prompts on a schedule](https://code.claude.com/docs/en/scheduled-tasks) for the comparison.

### Delegating to `Plan` from a scheduled agent

A scheduled hotel agent can also hand a research step to `Plan` in the middle of its run, before it prepares a draft or a proposal:

```
Use the Plan subagent to work out how to change the stale-lead rule without breaking the nightly job. House rules it must keep: never write to the PMS without approval; drafts only, never send.
```

Mechanics and limits from the docs:

- **Read-only.** Write and Edit are denied for `Plan`. It returns a summary to the agent that called it.
- **No `CLAUDE.md`.** Restate the rules that matter in the prompt. In a `-p` run you can also pass `--append-subagent-system-prompt "<your rules>"` (v2.1.205 or later), which adds the text to every sub-agent's system prompt.
- **Not guaranteed.** In a prompt, Claude decides whether to delegate. Plan mode is the guarantee that nothing is edited.
- **From a custom sub-agent.** A sub-agent can start `Plan` only if its `tools` list includes `Agent` (or it has no `tools` list). Nesting stops 3 layers below the main conversation by default.
- **Cost.** `Plan` uses the model of the main conversation and sends its own requests. They count toward the same usage limits. Keep the question narrow and name the files.

### What must never be delegated to `Plan`

- **Approval.** A plan is a proposal. Only the operator approves it.
- **Execution.** `Plan` cannot act. The agent that executes a step uses its own `AUTONOMY.md` level and limits. "The plan said so" is not a permission.
- **Decisions about money, complaints, legal, medical or safety matters.** `Plan` can list options. A person decides.
- **Live facts.** A plan can be hours old. The executing agent reads the booking, rate or mailbox again before it acts.
- **Changes to `AUTONOMY.md`, settings, hooks or permissions.** `Plan` can propose them. Only the operator makes them.

### Handoffs

- At Level 1 the plan is the run report. [`morning-routine-agent`](https://github.com/th1-ai/morning-routine-agent) can read `runs/plan-report.json` and show its "Needs you" section in the morning briefing.
- At Level 2 the calling agent saves the plan to `runs/handoff/plan-agent.md` in its own folder and replaces it on each run. The file has a `Plan date:` line, because the file name has no date. To approve the plan, fill in the `Approved:` line at the top of the file with the date and your name. The main session or [`general-purpose-agent`](https://github.com/th1-ai/general-purpose-agent) then executes it. `general-purpose-agent` reads `runs/handoff/plan-agent.md` and does nothing with a plan that has no approval or is more than 7 days old. The format is in [`AUTONOMY.md`](AUTONOMY.md), "The plan file".
- For a pure search question, use [`explore-agent`](https://github.com/th1-ai/explore-agent), not `Plan`.

Run reports can contain guest data. Add `runs/` to the `.gitignore` of the agent folder.

### Moving up a level

1. Run 7 days at the current level.
2. Read each run report. Check that "Needs you" was correct and that `permission_denials` shows nothing unexpected.
3. Change the one line `Level:` in `AUTONOMY.md`.
4. Read every run report for the first 5 runs at the new level before you approve a plan.

To stop everything, change the line to `Level: PAUSED` (or `Level: 0`).

### Subscription note

Scheduled `claude -p` runs on a personal Pro or Max plan are subject to Anthropic's usage policy, and the plan limits assume ordinary individual use; for heavy or business-critical schedules, use an API key or a team plan.

---

## Name collision: do not create your own agent called `Plan`

This repo does not define an agent, so it cannot collide with anything. The risk appears only if you add your own file such as `.claude/agents/plan.md` with `name: Plan`.

The docs state that a user or project sub-agent with the same name as a built-in **overrides the built-in** (their example is `Explore`). So a custom agent named `Plan` replaces the built-in one, and plan mode then uses your prompt, your tools and your model. If your file allows Write or Edit, plan-mode research is no longer read-only by definition of the agent.

The docs do not say whether the name match is case-sensitive. Treat `plan` as taken too.

If you want a custom planner (for example one that knows your PMS rules), give it a different name such as `hotel-change-planner`, and leave the built-in alone.

---

## Troubleshooting

- **Claude edited files instead of planning.** You were not in plan mode. Use `Shift+Tab` or `/plan`.
- **The plan ignores a house rule.** `Plan` does not read `CLAUDE.md`. Put the rule in the prompt.
- **`Plan` is never used.** Check that it is not disabled. It is off if your settings deny `Agent(Plan)` under `permissions.deny`, or if the environment variable `CLAUDE_CODE_DISABLE_EXPLORE_PLAN_AGENTS` is set to `1` (Claude Code v2.1.198 or later). Claude then explores the files directly instead.
- **`Plan` behaves differently from this page.** Look for a custom agent named `Plan` in `.claude/agents/` or `~/.claude/agents/` (see above), and check your version with `claude --version`. Built-in sub-agents change between releases; the official docs are the source of truth.

---

## File structure

```
plan-agent/
├── README.md                        # This file
├── AUTONOMY.md                      # Levels and limits for delegating to Plan
├── .claude/
│   └── settings.example.json        # Example permissions for a plan-only scheduled run
├── LICENSE
└── .gitignore
```

There is no agent file, no `.env.example`, and no placeholders to fill in. The agent lives inside Claude Code itself. `settings.example.json` is not loaded by Claude Code unless you pass it with `--settings`.

---

## Related agents

The other built-in sub-agents documented in this org:

- [`explore-agent`](https://github.com/th1-ai/explore-agent): fast read-only search
- [`general-purpose-agent`](https://github.com/th1-ai/general-purpose-agent): multi-step tasks that need both research and changes

The orchestrator that can read a scheduled plan report:

- [`morning-routine-agent`](https://github.com/th1-ai/morning-routine-agent)

Hotel-domain custom agents where a plan pass before changes pays off:

- [`email-processor-agent`](https://github.com/th1-ai/email-processor-agent)
- [`dynamic-pricing-agent`](https://github.com/th1-ai/dynamic-pricing-agent)
- [`crm-pipeline-agent`](https://github.com/th1-ai/crm-pipeline-agent)
- [`finance-filing-agent`](https://github.com/th1-ai/finance-filing-agent)

The full set of 21 agents: https://th1.ai/agent-stack

---

## Further reading

- Claude Code sub-agents (built-in agents, name precedence, how to disable one): https://code.claude.com/docs/en/sub-agents
- Permission modes, including plan mode: https://code.claude.com/docs/en/permission-modes
- Non-interactive runs (`claude -p`): https://code.claude.com/docs/en/headless
- Settings and permission rules: https://code.claude.com/docs/en/settings and https://code.claude.com/docs/en/permissions
- Usage policy and authentication: https://code.claude.com/docs/en/legal-and-compliance

---

## License

MIT. See [LICENSE](LICENSE).
