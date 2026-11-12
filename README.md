# Plan Agent

A pointer to the **`Plan` sub-agent built into [Claude Code](https://code.claude.com/docs/en/overview)**. There is nothing to install. This repo documents when and how to use it from a hotel-operations workflow.

> **Note:** Unlike the other th1-ai agents, `Plan` is not a custom sub-agent definition. It ships with Claude Code. There is no agent file to copy, and nothing to configure.

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
├── README.md       # This file: the whole repo, essentially
├── LICENSE
└── .gitignore
```

There is no `.claude/` folder, no `.env.example`, and no placeholders to fill in. The agent lives inside Claude Code itself.

---

## Related agents

The other built-in sub-agents documented in this org:

- [`explore-agent`](https://github.com/th1-ai/explore-agent): fast read-only search
- [`general-purpose-agent`](https://github.com/th1-ai/general-purpose-agent): multi-step tasks that need both research and changes

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

---

## License

MIT. See [LICENSE](LICENSE).
