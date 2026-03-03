# Profiles

Sandboxec profiles for running CLI tools with a tighter blast radius on Linux.

This repository contains YAML policy files you can pass to [**sandboxec**](https://github.com/sandboxec/sandboxec), a lightweight command sandbox built on Linux Landlock. It restricts filesystem and TCP access for a wrapped command and all of its child processes.

> **sandboxec** is command-level containment, not a full VM/container replacement.

## Requirements

- Linux kernel `>= 5.13` (Landlock enabled)
- **sandboxec** installed
- For TCP `bind/connect/bind_connect` rules (`net`), newer Landlock support is needed (ABI v4+, commonly kernel `>= 6.7`)

## Quick start

Run a command with a profile from this repo:

```bash
sandboxec --config profiles/<group>/<profile>.yaml -- your-command
```

## How profiles are structured

Each profile uses Sandboxec YAML keys such as:

- `abi` — target Landlock ABI (this repo uses `6`)
- `ignore-if-missing` — skip missing paths instead of failing
- `unsafe-host-runtime` — broaden runtime/library access for host-linked tools
- `fs` — allow-list of filesystem rights (`r`, `rx`, `w`, `rw`, `rwx`)
- `net` — allow-list of TCP rights (`b`, `c`, `bc`) by port

Rules are allow-list based: if it is not explicitly allowed, it is denied.

## Tuning a profile

If a command fails with `permission denied`:

- Add only the missing runtime paths or ports required by the command.
- Retry and keep the profile as narrow as possible.
- Use `--unsafe-host-runtime` only when host-linked runtime access is required.

> [!TIP]
> Use `strace` to find denied file/network accesses while tuning rules.

```bash
sandboxec --config profiles/<group>/<profile>.yaml -- strace -f -e trace=file,network your-command
```

Useful fallback during compatibility issues:

```bash
sandboxec --best-effort --config profiles/<group>/<profile>.yaml -- your-command
```

## Contributing

For profile authoring standards and PR expectations, see [CONTRIBUTING.md](./.github/CONTRIBUTING.md).

## License

Licensed under the [DO WHAT THE FUCK YOU WANT TO PUBLIC LICENSE](./COPYING) Version 2.