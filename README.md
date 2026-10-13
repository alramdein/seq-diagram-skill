# seq-diagram

A Claude Code skill that writes Mermaid sequence diagrams of **business flows** into
`docs/flows/`, and regenerates them from a git diff instead of re-reading your repo.

## The problem

Ask an LLM to diagram a flow and you get one of two bad outcomes:

1. **Too detailed.** Every `validateEmail`, `setLoading`, `parseBody` becomes a step.
   Forty lines nobody reads.
2. **Unstable.** Run it again next month and participants reorder, labels reword, and
   the diff is unreadable — so you stop trusting the diff, and the doc rots.

Both come from the same cause: the model is deciding what goes in the diagram, from a
blank page, every time.

## The fix

**A rule picks the steps, not the model.** A step exists only when control leaves the
service — HTTP call, DB query, cache op, cookie write, third-party SDK, queue publish.
Helpers, validation, logging and mapping are invisible. Hard cap of 15 steps; over the
cap means it's two flows, so split it rather than shrink the labels.

**The committed diagram is the cache.** Each file carries the SHA it was traced from
and the list of files it was traced through:

```yaml
---
flow: sign-in
generated_from: a1b2c3d
participants: [FE, API, Auth, DB, Cache]
sources:
  - web/src/views/LoginForm.tsx
  - services/api/src/domain/session/sessionService.ts
---
```

Regenerating runs `git diff a1b2c3d..HEAD -- <sources>` first. Empty diff means the flow
is unchanged and **not one source file gets opened**. A non-empty diff means only the
changed hunks get read, and only the affected rows get patched — participant order and
untouched lines stay byte-identical, so the diff shows the real change.

First trace is expensive, because something actually has to read the code. Every
regeneration after that is close to free.

## Citations are mandatory

Every step cites `file:line` in a table under the diagram. No citation means it wasn't
verified, so it doesn't get drawn.

This is the part that catches hallucinated hops — a wrong arrow renders just as nicely
as a right one, and nothing else will tell you. On its first real run against a
multi-service auth stack, the citation rule turned up three things a careful
hand-traced diagram had gotten wrong, including a cross-service call that wasn't in the
flow at all.

## Two levels

| Mode | Shows | Use for |
| ---- | ----- | ------- |
| `flow` (default) | boundary crossings, all participants | onboarding, docs, "how does X work" |
| `--detail <participant>` | one participant's internals, everyone else collapsed | changing that service's code |

Detailing two participants at once is what produces the unreadable diagram. The skill
refuses to.

## Install

```bash
git clone git@github.com:alramdein/seq-diagram-skill.git
mkdir -p <your-repo>/.claude/skills/seq-diagram
cp seq-diagram-skill/SKILL.md <your-repo>/.claude/skills/seq-diagram/
```

Project-level so the repo's other contributors get it on clone. Use
`~/.claude/skills/seq-diagram/` instead if you want it everywhere.

## Usage

```
diagram the sign-in flow
/seq-diagram sign-in
/seq-diagram sign-in --detail auth-service
regen sign-in diagram
```

Output lands in `docs/flows/sign-in.md`. GitHub renders Mermaid natively, so the file
is readable where it sits.

## Deliberately not included

No CI step, no pre-commit hook, no staleness checker. Regeneration is on demand. The
tradeoff is that nothing tells you a diagram went stale — you regen when you suspect it,
or when you're onboarding someone. If you want the nag, a script that checks each cited
`file:line` still contains its cited symbol is about thirty lines.

## License

MIT
