---
name: sandboxec-profiles
description: Builds sandboxec YAML policies from a concrete `your-command` workflow on Linux using Landlock. Use when deriving least-privilege fs/net rules from real command behavior, fixing permission denied errors, and documenting rule-by-rule rationale.
compatibility: Linux with sandboxec installed; intended for repositories that store sandboxec profile YAML files and documentation.
license: WTFPL
metadata:
  domain: sandboxing
  focus: command-first-policy-synthesis
---

# Sandboxec profile authoring skill

This skill helps you create and maintain practical, least-privilege `sandboxec` profiles for CLI tools.

Core objective: build policy from `your-command` itself (what it reads, writes, executes, and where it connects), not from generic templates.

Primary references in this repo:
- [README.md](README.md)
- [.github/CONTRIBUTING.md](.github/CONTRIBUTING.md)
- Existing examples in `agents/*.yaml`

## When to use this skill

Use this skill when you need to:
- add a new profile for a CLI tool
- tune an existing profile after `permission denied` failures
- reduce an overly broad profile back to minimal permissions
- explain and document why specific `fs` or `net` rules are required

## Expected output

Produce:
1. a minimal YAML policy file (or patch) for the exact `your-command`
2. a rule-to-reason map (each `fs`/`net` entry tied to observed behavior)
3. verification commands showing success under sandbox and failures without required rules

If documentation or usage changes, update [README.md](README.md) accordingly.

## Repo conventions to follow

- Use `abi: 6` unless explicitly told otherwise.
- Prefer `--config <file>` with YAML profiles for repeatable policies; use CLI flags (`--fs`, `--net`) only for quick experiments.
- `--config` may point to local YAML or remote `http(s)` YAML; prefer local repo files for repeatable tuning.
- `--named-config <name>` (or `-C <name>`) resolves upstream profiles from `sandboxec/profiles` and is useful for baseline comparison.
- Prefer `ignore-if-missing: true` only for optional paths.
- Use `restrict-scoped: true` only when scoped IPC restrictions are required and environment supports ABI v6+.
- Keep `unsafe-host-runtime: true` only when host-linked runtime access is truly needed.
- Keep rules allow-list and narrow:
  - choose `r`/`rx` before `rw`
  - avoid broad writable paths
  - allow only required TCP ports (`net` entries like `c:443`)
- Preserve YAML formatting/style used in existing profiles.
- Avoid unrelated refactors or formatting-only edits.

## Authoring workflow

1. Normalize the target invocation.
  - define one exact command string as `your-command`
  - include representative arguments, env vars, working directory, and expected network behavior
  - confirm execution mode: `run` (normal command sandboxing) or `mcp` (MCP-focused runtime flow)
2. Build a minimal first policy for `your-command`.
3. Add only essential filesystem paths for that command:
   - workspace writes typically `rw:$PWD`
   - tool-specific state/config/cache paths under `$HOME`
   - runtime/CA/proc files as read-only only when observed needed
4. Add only required network ports used by that command in `net`.
5. Test `your-command` under sandbox:

```bash
sandboxec --config profiles/<group>/<profile>.yaml -- your-command
```

6. If denied, trace accesses for that exact command and tighten iteratively:

```bash
sandboxec --config profiles/<group>/<profile>.yaml -- strace -f -e trace=file,network your-command
```

7. Re-test after each permission change; remove any rule not proven necessary for `your-command`.

## Command-first rule derivation checklist

For each candidate permission, require all three:
- Evidence: observed access from run logs/strace/error output.
- Necessity: command fails without it.
- Minimality: narrowest right and narrowest path/port that still works.

If any item is missing, do not keep the rule.

## CLI options and rights reference

Use option names and rights exactly as supported by current `sandboxec --help`:

- config and rule inputs:
  - `--config <path-or-url>`
  - `--named-config <name>` (`-C <name>`)
  - `--fs RIGHTS:PATH` (repeatable)
  - `--net RIGHTS:PORT` (repeatable)
- behavior flags:
  - `--abi <1-6>` (`0` means default)
  - `--best-effort`
  - `--ignore-if-missing`
  - `--restrict-scoped` (ABI v6+)
  - `--unsafe-host-runtime`
  - `--mode run|mcp`

Mode note:
- In `--mode mcp`, wrapped command arguments are not accepted.

Rights aliases:
- filesystem:
  - `read` = `r`
  - `read_exec` = `rx`
  - `write` = `w`
  - `read_write` = `rw`
  - `read_write_exec` = `rwx`
- network:
  - `bind` = `b`
  - `connect` = `c`
  - `bind_connect` = `bc`

Prefer short aliases (`r`, `rx`, `rw`, `c`) in profiles for consistency with this repository.

## Config behavior and precedence

Keep these semantics in mind when testing profile changes:

- config lookup (if neither `--config` nor `--named-config` is provided):
  1. `$XDG_CONFIG_HOME/sandboxec/sandboxec.yaml|yml`
  2. `$HOME/.config/sandboxec/sandboxec.yaml|yml`
  3. `/etc/sandboxec/sandboxec.yaml|yml`
- precedence:
  - `--config` and `--named-config` are mutually exclusive
  - scalar CLI flags override YAML scalar values
  - if `--fs` and/or `--net` are provided via CLI, they replace config lists
  - if not provided, `fs`/`net` rules come from YAML config

For reproducible profile development in this repo, prefer explicit `--config <profile.yaml>`.

## Review checklist

Before finalizing changes, verify:
- the exact `your-command` starts and runs as intended
- no unnecessary writable paths were added
- no unnecessary outbound ports were allowed
- unusual allowances are documented in comments or PR notes
- README paths/examples remain accurate after structural changes

## Final response format

When reporting results, include:
1. `your-command` used for validation
2. added/changed/removed rules
3. one-line reason per rule
4. test command(s) and observed outcome

## Profile template

Use this as a baseline, then prune/add narrowly:

```yaml
abi: 6
ignore-if-missing: true
unsafe-host-runtime: true
fs:
  - rw:$PWD
  - rw:$HOME/.tool/
  - r:/etc/hosts
  - r:/etc/resolv.conf
  - r:/etc/ssl/certs
  - r:/proc/
  - rw:/dev/null
net:
  - c:443
```

## Decision guidance for common keys

- `ignore-if-missing`: enable when a path is optional across distros/setups.
- `restrict-scoped`: enable only for workflows that need tighter scoped IPC restrictions and are confirmed on ABI v6+ kernels.
- `best-effort`: use for compatibility checks on older kernels; do not rely on it as a permanent substitute for correct capability targeting.
- `unsafe-host-runtime`: enable for host-linked runtimes that probe many host files; disable if static/self-contained runtime works without it.
- `mode`:
  - `run` for standard command wrapping
  - `mcp` for MCP-oriented executions where the toolchain expects MCP mode behavior
- `fs` rights:
  - `r` for read-only files/directories
  - `rx` for executables/readable directories when execute is required
  - `w` for narrow write-only sinks (rare; prefer explicit path)
  - `rw` only for explicit state/cache/work dirs
  - `rwx` only when execute on writable content is demonstrably required
- `net` rights:
  - `c:<port>` for outbound connects
  - `b:<port>` only when the tool must listen
  - `bc:<port>` only when both bind and connect are required

## Notes for agent behavior

- Prefer minimal diffs in existing files.
- Do not broaden permissions “just in case.”
- When uncertain, choose the narrower permission and iterate from observed failures.
- Include a concise permission-by-permission rationale in your final summary.

## Troubleshooting cues

- `invalid fs/net rights`: verify exact rights spelling (`r`, `rx`, `rw`, `b`, `c`, `bc`).
- Landlock/ABI failures: check kernel capability first (fs v1+, net v4+, scoped restrictions v6+).
- If uncertain whether failure is unsupported feature vs missing permission, retry with `--best-effort` to isolate compatibility issues.
- `permission denied` under least-privilege tuning usually means missing runtime reads (for example `/usr`, `/lib*`, certs, or selected `/etc` files) rather than missing writes.