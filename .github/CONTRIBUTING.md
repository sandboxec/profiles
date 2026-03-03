# Contributing

Thanks for helping improve these Sandboxec profiles.

## Scope

This repository contains practical, reusable **sandboxec** YAML profiles for CLI workflows on Linux.

## Contribution principles

- Keep profiles least-privilege by default.
- Add only the minimum `fs` and `net` rules needed.
- Prefer profile-specific paths over broad paths.
- Document why unusual allowances are required.
- Avoid adding unrelated formatting or refactors in the same change.

## Editing profiles

When changing a profile (`*.yaml`):

1. Start from the smallest allow-list that still works.
2. Use narrow filesystem rights first (`r` or `rx`) before `rw`.
3. Limit network access to required ports only.
4. Keep `unsafe-host-runtime: true` only when needed for host-linked runtimes.
5. Use `ignore-if-missing: true` only for optional paths.

## Use the skill guide

For profile authoring and tuning workflow, follow [SKILL.md](../SKILL.md).

- Treat it as the command-first playbook for deriving rules from `your-command`.
- Include rule-by-rule rationale and validation commands in your PR notes.

## Group folders

Profiles may be organized into group folders (for example, `agents/`) when it improves clarity.

- Use a group folder only when at least 2 related profiles share the same use-case.
- Use short, lowercase folder names based on use-case.
- Keep profile filenames stable when moving them into a folder.
- Update all path references in `README.md` when introducing or changing groups.
- In the PR description, explain why the new group is needed.

## Validation checklist

Before opening a PR:

- The target command starts successfully with the updated profile.
- No extra writable paths were added without clear need.
- No unnecessary outbound ports were allowed.
- YAML is valid and consistently formatted.
- README remains accurate if behavior/usage changed.

## Pull request guidance

Please include:

- What command/workflow the profile is for.
- Which rules were added/removed/changed.
- Why each new permission is required.
- Any kernel or distro assumptions.
- A short test command used to verify behavior.

## Code of conduct

Be respectful and constructive in discussions and reviews.
