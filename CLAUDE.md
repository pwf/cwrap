# cwrap

Podman wrapper around the `claude` CLI. Each instance gets an isolated home directory so Claude's config/history/MCP servers live at the normal `$HOME/.config/claude/` path. Only explicitly allowed dirs are visible inside the container.

## Files

- `cwrap` — main bash script (the entire tool)
- `Containerfile` — canonical template; copy per-instance to `~/.config/cwrap/instances/<name>/Containerfile`
- `examples/` — ready-to-copy config files

## Runtime config layout

```
~/.config/cwrap/
├── config                          # global: default_instance=<name>
└── instances/<name>/
    ├── Containerfile               # per-instance image (build context is this dir)
    ├── cwrap.conf                  # CWRAP_DIRS=(...) — sourced as bash
    └── home/                       # mounted as /home/cwrap inside container
        └── .config/claude/         # claude's normal config path
```

## Usage

```
cwrap build [-i <instance>]              # build image from instances/<name>/Containerfile
cwrap [-i <instance>] [dir...]           # drop into bash (mounts only)
cwrap [-i <instance>] [dir...] -- <args> # run: claude --add-dir <dirs> <args>
```

## Key design decisions

- **Podman only** — no Docker fallback; `--userns=keep-id` for rootless UID mapping
- **Per-instance Containerfile** — lives in the instance dir, build context is that dir
- **No PWD mount** — only `/home/cwrap` + dirs from `CWRAP_DIRS`/positional args are visible
- **bash by default** — `CMD ["bash"]`; claude only runs when passthrough args are given
- **`CWRAP_DIRS`** — mounts dirs and stages `--add-dir` flags, but does not auto-launch claude
- **Native claude install** — `curl -fsSL https://claude.ai/install.sh | bash` (npm is deprecated)
- **Non-root user** `cwrap` — home is `/home/cwrap`; binary at `/home/cwrap/.local/bin/claude`; PATH set in image
- **`ANTHROPIC_*`** env vars forwarded from host
- **`:z`** on all volumes — SELinux relabeling, no-op elsewhere
