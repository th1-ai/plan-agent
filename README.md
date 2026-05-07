# Plan Agent

A pointer to the **`Plan` sub-agent built into [Claude Code](https://claude.ai/code)**. There is nothing to install — this repo exists to document when and how to use it from a hotel-operations workflow.

> **Note:** Unlike the other th1-ai agents, `Plan` is not a custom sub-agent definition. It ships with Claude Code. There is no `agent.md` to copy.

---

## What it does

`Plan` is a software-architect sub-agent. Given a problem, it:

- Reads the relevant parts of the codebase to ground itself in current state
- Returns a step-by-step implementation plan
- Identifies the critical files and call sites that will need to change
- Calls out architectural trade-offs, risks, and likely failure modes
- Stops short of writing code — its output is a plan, not a patch

The point is to think *before* changing anything, especially in workflows where a wrong edit costs real time (live PMS bookings, finance logs, customer-facing email drafts).

---

## What it is **not**

`Plan` is not for:

- **Searching** — use [`explore-agent`](https://github.com/th1-ai/explore-agent) for "where is X defined"
- **Writing the code** — `Plan` returns a plan; the main agent (or you) executes it
- **Trivial one-line fixes** — overkill, just edit the file
- **Open-ended research** — use [`general-purpose-agent`](https://github.com/th1-ai/general-purpose-agent) for "should we even do this?"

---

## When Claude Code picks it automatically

Claude Code routes to `Plan` (or invokes it under the hood) when the main agent's plan includes a step like:

- "Design the change before editing"
- "Lay out the steps to add feature X"
- "Decide how to refactor this safely"
- "What's the smallest change that fixes this without breaking Y?"

It is the right call any time the work touches multiple files, has non-obvious sequencing, or needs to preserve a critical invariant (no double-charging guests, no mutating live reservations without approval, etc.).

---

## How to invoke it explicitly

From inside a Claude Code session:

```
> Use the Plan subagent to design how we'd add a stale-lead chase to the CRM pipeline without breaking the existing nightly cron.
```

Or, more naturally:

```
> Before you write any code, think through how to add price-drop alerts to the agent-notification flow.
```

Claude Code will normally pick `Plan` itself when the request is design-shaped rather than search-shaped.

---

## Suggested usage with hotel agents

Use `Plan` before:

- Adding a new skill to an existing agent (e.g. extending `email-triage-agent` to handle a new sender pattern)
- Wiring a new tool (e.g. swapping the PMS adapter from one provider to another)
- Refactoring an SOP that several agents depend on
- Any change that touches finance logging or live PMS writes

Skip `Plan` for:

- Editing copy in a draft email
- Adjusting a single threshold or label
- Changes confined to one file with obvious shape

---

## File structure

```
plan-agent/
├── README.md       # This file — the whole repo, essentially
├── LICENSE
└── .gitignore
```

There is no `.claude/` folder, no `.env.example`, and no `agent.md`. The agent lives inside Claude Code itself.

---

## Related agents

This sub-agent is one of three built-ins documented in this org:

- [`explore-agent`](https://github.com/th1-ai/explore-agent) — fast read-only search
- [`general-purpose-agent`](https://github.com/th1-ai/general-purpose-agent) — default fallback for multi-step research

Hotel-domain custom agents that benefit from a `Plan` pass before changes:

- [`email-processor-agent`](https://github.com/th1-ai/email-processor-agent)
- [`dynamic-pricing-agent`](https://github.com/th1-ai/dynamic-pricing-agent)
- [`crm-pipeline-agent`](https://github.com/th1-ai/crm-pipeline-agent)
- [`finance-filing-agent`](https://github.com/th1-ai/finance-filing-agent)

---

## Further reading

- Official Claude Code sub-agents docs: https://docs.claude.com/en/docs/claude-code/sub-agents

---

## License

MIT — see [LICENSE](LICENSE).
