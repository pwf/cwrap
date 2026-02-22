# cwrap

Wrapper over `claude` CLI that isolates instances (personal, work, …) via containers. Each instance gets its own home directory (bind-mounted from the host) so claude finds config, history, MCP servers at the default `$HOME/.config/claude/` path without any overrides. Only explicitly granted directories are visible inside the container. Requires [Podman](https://podman.io/) for OS-level isolation.

## Usage

```
cwrap build [-i <instance>]
cwrap [-i <instance>] [dir...]
cwrap [-i <instance>] [dir...] -- <claude args...>
```

- `-i/--instance` — which instance; optional if `default_instance` set in config
- `[dir...]` — dirs to bind-mount and pass as `--add-dir` to claude
- `build` — first positional arg: build the container image
- `-- <args>` — everything after `--` passes through to claude

## Config

```
~/.config/cwrap/
├── config                  # global: default_instance=personal
└── instances/
    └── <name>/
        ├── Containerfile   # per-instance image
        ├── cwrap.conf      # instance config: CWRAP_DIRS=(~/projects)
        └── home/           # mounted as /home/cwrap inside container
            └── .config/claude/ # claude's default config path ($HOME/.config/claude/)
```

Sourced as bash. `<name>/home/` is mounted as `/home/cwrap` inside the container. Claude uses `$HOME/.config/claude/` as normal — no overrides needed.

## Setup

Copy example configs to get started:

```bash
mkdir -p ~/.config/cwrap/instances/personal
cp examples/config ~/.config/cwrap/config
cp examples/instances/personal/cwrap.conf ~/.config/cwrap/instances/personal/cwrap.conf
cp examples/instances/personal/Containerfile ~/.config/cwrap/instances/personal/Containerfile
```

Then build the image:

```bash
cwrap build -i personal
```

## Examples

```bash
# build default image
cwrap build -i personal

# run with default instance (reads default_instance=personal from config)
cwrap

# run work instance, add extra dir
cwrap -i work ~/tmp

# pass claude flags through
cwrap -i work -- --resume
```

## How it works

- **`--userns=keep-id`** — rootless Podman: host UID maps into container, file ownership works.
- **`:z`** — SELinux relabeling; no-op elsewhere.
- **`ANTHROPIC_*`** env vars from the host are forwarded into the container automatically.
- **No PWD mount** — only `/home/cwrap` and explicitly granted dirs are visible inside the container.
