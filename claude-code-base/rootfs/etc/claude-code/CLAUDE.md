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

If the Containerfile does not exist yet, create it as described under
"Writing the Containerfile" and then read `.claude-code-container/config`.
The launcher builds the Containerfile only when `config` sets
`claude_code_podman_image_base`, and it refuses to start while the
Containerfile exists and that variable is unset. If the variable is unset or
`config` does not exist, ask the user to add this line to
`.claude-code-container/config` before they start the next session:

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

## Writing the Containerfile

`/usr/local/share/claude-code-base/Containerfile` is the Containerfile of
`claude-code-base`, the image that every project image is built on. Read it
before you create or edit `.claude-code-image/Containerfile`. It shows what
is already installed, and it is the reference for style and approach: the
layout of a `run` step, an apt step with its cache mounts and package list,
a tool that Debian does not package (hurl), and files copied from `rootfs/`.

A project Containerfile starts with these two lines in place of the `from`
line of the base:

```
arg claude_code_podman_image_base
from localhost/${claude_code_podman_image_base}:latest
```

- Keep the two lines as they are. The launcher passes the name of the base
  image as a build argument.
- Add only what the project needs. Do not repeat what the base Containerfile
  already did, such as removing `docker-clean`, creating the user or setting
  the entrypoint.
- Build steps run as root, so `sudo` is not needed in the Containerfile.
- Use the cache mounts as the base does, with the same ids: both apt mounts
  on every step that runs apt, and the downloads mount, with a subdirectory
  named after the tool, on a step that downloads a file.
- The build context is `.claude-code-image/`. Keep files for the image in
  `.claude-code-image/rootfs/` at the path they have in the image, and copy
  each one with its own instruction, as the base does.
