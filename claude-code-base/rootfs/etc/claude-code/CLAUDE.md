# Container environment

You are running in a disposable Podman container (Debian trixie) that the
`claude-code` launcher script started on the user's machine. The container is
deleted when the session ends.

- The project directory is mounted from the host at the same path, and
  `~/.claude` is a persistent volume shared by all projects. Everything else
  is lost when the session ends, including packages you install and files
  elsewhere in the home directory.
- You are the user `claude` and have passwordless `sudo`.
- There is no Podman or Docker in the container, so the image cannot be built
  or tested from here.
- `~/.config/git` is the user's git configuration, mounted read-only.

## Launcher files in the project

Both are in the project root, which is the directory the session started in.

- `.claude-code-image/Containerfile` defines the image for this project. You
  may create and edit it. The launcher builds it on the host each time a
  session starts.
- `.claude-code-container/` is mounted read-only. Its `config` file is sourced
  by the launcher on the host and holds launcher settings and extra
  `podman run` options. Only the user can change it.

## When a tool is missing

1. Install it for the current session so that you can continue. For Debian
   packages run `sudo apt-get update` and then `sudo apt-get install`; the
   image ships without package lists, so the update is required.
2. Add it to `.claude-code-image/Containerfile` so that later sessions have
   it.
3. Tell the user what you added. It takes effect when they start the next
   session, so ask them to restart if you could not install it in step 1.

If the Containerfile does not exist yet, create it from the example below and
then read `.claude-code-container/config`. The launcher builds the
Containerfile only when `config` sets `claude_code_podman_image_base`, and it
refuses to start while the Containerfile exists and that variable is unset.
If the variable is unset or `config` does not exist, ask the user to add this
line to `.claude-code-container/config` before they start the next session:

```
claude_code_podman_image_base='claude-code-base'
```

Once the variable is set, do not delete the Containerfile: the launcher also
refuses to start when it is missing.

Options for `podman run`, such as published ports, volumes or environment
variables, can only be set in `config`. When you need one, give the user the
line to add, for example:

```
claude_code_podman_args+=(--publish='127.0.0.1:8080:8080')
```

## Containerfile example

```
arg claude_code_podman_image_base
from localhost/${claude_code_podman_image_base}:latest

run \
  --mount=type=cache,id=trixie-apt-cache,target=/var/cache/apt,sharing=locked \
  --mount=type=cache,id=trixie-apt-lists,target=/var/lib/apt/lists,sharing=locked \
  <<'eof'
# install project packages
set -o errexit

apt-get update

DEBIAN_FRONTEND=noninteractive \
apt-get --no-install-recommends --assume-yes install \
  'netcat-openbsd' \
  'postgresql-client'

eof
```

- Keep the first two lines as they are. The launcher passes the name of the
  base image as a build argument.
- Build steps run as root, so `sudo` is not needed in the Containerfile.
- Keep both cache mounts on every step that runs apt. They hold the package
  lists and downloaded packages between builds.
- Follow the style of the example: lowercase instructions, one heredoc per
  `run` step that starts with a comment and `set -o errexit`, and quoted
  package names, one per line, in alphabetical order.
- For tools that are not packaged for Debian, add another `run` step in the
  same style.
- The build context is `.claude-code-image/`. Keep files for the image in
  `.claude-code-image/rootfs/` at the path they have in the image, and copy
  each one with its own instruction that sets the mode and names the
  destination directory, for example
  `copy --chmod=644 'rootfs/etc/example.conf' '/etc/'`.
