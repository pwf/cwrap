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
cwrap [-i <instance>] [dir...] -- <args> # run: claude <args> (codex: --add-dir <dirs> <args>)
```

## Key design decisions

- **Podman only** — no Docker fallback; `--userns=keep-id` for rootless UID mapping.
- **One pod per instance** (`cwrap-<instance>`, `--share pid` only) — claude's peer sessions register as `~/.claude/sessions/<pid>.json` and are checked by PID, so the containers of an instance share one PID namespace; separate containers were all PID 1. Cost: sessions of one instance see each other's processes, including `/proc/<pid>/environ` and `/proc/<pid>/root`. Not `--pid=host`: that exposes every host process of the user
- **Per-instance Containerfile** — lives in the instance dir, build context is that dir
- **No PWD mount** — only `/home/cwrap` + dirs from `CWRAP_DIRS`/positional args are visible
- **bash by default** — `CMD ["bash"]`; claude only runs when passthrough args are given
- **`CWRAP_DIRS`** — mounts dirs, but does not auto-launch claude
- **No `--add-dir` for claude** — since 2.1.326 claude asks to trust every `--add-dir` folder on each start and never saves the answer ([#110642](https://github.com/anthropics/claude-code/issues/110642)); the mount is the trust decision. Keep related dirs inside the working dir: they are mounted and accessible without extra entries. Codex keeps `--add-dir`: it sets the writable roots of `--sandbox workspace-write`
- **Native claude install** — `curl -fsSL https://claude.ai/install.sh | bash` (npm is deprecated)
- **Non-root user** `cwrap` — home is `/home/cwrap`; claude installed system-wide at `/usr/local/bin/claude` (so home mount doesn't shadow it)
- **`ANTHROPIC_*`** env vars forwarded from host
- **`:z`** on all volumes — SELinux relabeling, no-op elsewhere
