# Contributing

Thanks for helping improve the SWML Claude skill.

## Ground rules

- Keep diffs small and focused — one concern per pull request.
- Back behavioral claims with a citation: official SignalWire docs, a
  support ticket, or a reproducible call flow. This skill teaches
  Claude; an unsourced "fact" here becomes a confidently wrong agent.
- `schema.json` and `swml_reference.md` must agree. If a change touches
  a value in one, check the other.

## Workflow

1. Fork and branch off `main`.
2. Open a pull request with the evidence for the change in the body.
3. Maintainers may fold a change in on their side rather than merging
   the PR directly, with authorship preserved — both count as accepted.

## Reporting problems

Open an issue with the SWML snippet, the expected and actual behavior,
and where the skill's guidance led an agent astray.
