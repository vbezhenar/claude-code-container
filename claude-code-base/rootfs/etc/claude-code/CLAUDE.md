# Container environment

You are running in a disposable Podman container (Debian trixie) that the
`claude-code` launcher script started on the user's machine.

- The project directory is mounted from the host at the same path, and
  `~/.claude` is a persistent volume shared by all projects. Everything else
  is lost when the session ends, including packages you install and files
  elsewhere in the home directory.
- You are the user `claude` and have passwordless `sudo`.

## Launcher files in the project

Both are in the project root, which is the directory the session started in.

- `.claude-code-image/Containerfile` defines the image for this project. You
  may create and edit it. The launcher builds it on the host each time a
  session starts. There is no Podman or Docker in the container, so it
  cannot be built or tested from here, and a build that fails stops the next
  launch.
- `.claude-code-container/` is mounted read-only. Its `config` file is sourced
  by the launcher on the host and holds launcher settings and extra
  `podman run` options. Only the user can change it.

Options for `podman run`, such as published ports, volumes or environment
variables, can only be set in `config`. When you need one, give the user the
line to add, for example:

```
claude_code_podman_args+=(--publish='127.0.0.1:8080:8080')
```

## When a tool is missing

1. Install it for the current session so that you can continue. For Debian
   packages run `sudo apt-get update` first: the image ships without package
   lists.
2. Add it to `.claude-code-image/Containerfile` so that later sessions have
   it.
3. Tell the user what you added.

## Writing the Containerfile

`/usr/local/share/claude-code-base/Containerfile` is the Containerfile of
`claude-code-base`, the image that every project image is built on. Read it
before you create or edit `.claude-code-image/Containerfile`. It shows what
is already installed, and it is the reference for style and approach: write
your steps the same way, with the same cache mounts, but do not repeat what
it has already done, such as removing `docker-clean`.

A project Containerfile starts with these two lines. Keep them as they are:
the launcher passes the name of the base image as a build argument.

```
arg claude_code_podman_image_base
from localhost/${claude_code_podman_image_base}:latest
```

The build context is `.claude-code-image/`. Keep files for the image in
`rootfs/` there, at the path they have in the image.

The launcher needs the Containerfile and `claude_code_podman_image_base` in
`config` together: it builds the Containerfile only when the variable is
set, and it refuses to start when one is there without the other. After you
create the Containerfile, read `.claude-code-container/config`. If it does
not exist or does not set the variable, ask the user to add this line to it
before they start the next session:

```
claude_code_podman_image_base='claude-code-base'
```
