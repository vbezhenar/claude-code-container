# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A personal tool that runs Claude Code inside a disposable rootless Podman
container. `claude-code` is the launcher, run on the host from a project's
root. `claude-code-base/` is the image every session starts from, and
`claude-code-base-java/` is an intermediate base for Java projects. The
launcher resolves its own real location to find the image directories, so it
is meant to be symlinked into `PATH`.

Sessions that work on this repository run inside the container it produces.
Nothing can be exercised from inside: there is no Podman, and changes to the
launcher or the base image only take effect on the user's next launch. The
base image is rebuilt at the start of every launch in every project and a
failed build aborts the launch, so a mistake in `claude-code-base/Containerfile`
or in the launcher blocks all sessions until it is fixed on the host.

## Checking changes

- `bash -n claude-code && shellcheck claude-code` prints nothing and exits 0.
  The `# shellcheck source=/dev/null` directive above the `source` line is
  there because the config belongs to whichever project the launcher runs in,
  so ShellCheck has no file to follow.
- Launcher logic: put stub `podman` and `timedatectl` executables first on
  `PATH` that print their arguments (the `podman` stub has to write an id to
  the `--iidfile` path), then run a copy of the launcher, placed in a scratch
  directory, in a scratch project directory for each combination of `config`
  contents and image directory. Do not run the launcher of this checkout with
  stubs: a stub pull that succeeds touches the real `.pull-stamp`, and the
  host then skips the Debian pull for a day.
- Containerfile syntax: `buildah` and `netavark` come from this project's
  image (`.claude-code-image/Containerfile`); install them with apt if the
  session runs on the plain base. `sudo buildah --storage-driver=vfs build`
  handles every instruction except `run` on a `from scratch` image, including
  `copy` heredocs, so the files they write and the resulting entrypoint can be
  inspected with `buildah from` and `buildah mount`. `run` steps cannot
  execute in the nested container; run their shell body directly instead.
- Launching from `claude-code-base/` or `claude-code-base-java/` uses the
  `config` there, which replaces the entrypoint with `bash`: a debug shell in
  that image. The Java directory is launched as a project on its own base,
  and its `.claude-code-image/Containerfile` adds Temurin 21, so the shell
  has a working `mvn`.

## What a launch runs

| `.claude-code-container/config` sets | Result |
|---|---|
| nothing | Runs `localhost/claude-code-base:latest`. Refuses to start if `.claude-code-image/Containerfile` exists. |
| `claude_code_podman_image` | Runs `localhost/<image>:latest` as it is; nothing is built for it. Same refusal as above. |
| `claude_code_podman_image_base` | Requires `.claude-code-image/Containerfile`. Builds `<launcher dir>/<base>/` first unless the base is `claude-code-base`, then builds the project image with the base name as build argument and runs it by image id. |
| both | The same, with the project image tagged `localhost/<image>:latest`. |

## Things that have to change together

- The agent instructions embedded in `claude-code-base/Containerfile`
  (installed as `/etc/claude-code/CLAUDE.md`) describe the launcher: file and
  variable names, the two refusal checks and the Containerfile example. Update
  them whenever the launcher's behaviour changes. Their example contains a
  line that is only `eof`, so the `copy` heredoc wrapping the text needs
  another delimiter (`eof1`).
- Project Containerfiles depend on the build argument name
  `claude_code_podman_image_base` and on the tag format
  `localhost/<name>:latest`.
- The entrypoint, `CLAUDE_CONFIG_DIR` and `LANG` live in the image, not in the
  launcher. `CLAUDE_CONFIG_DIR` points into the volume so that `.claude.json`
  persists as well.
- An intermediate base image is a directory next to the launcher that is named
  after the base (for example `claude-code-base-java/`) and whose
  Containerfile starts from `localhost/claude-code-base:latest`.
- The launcher pulls `docker.io/library/debian:trixie` by name. It has to be
  the image in the `from` line of `claude-code-base/Containerfile`.

## Design decisions

- **Disposable container, image rebuilt on every launch.** The Containerfile
  stays the only description of the environment, and an agent with sudo and no
  permission prompts starts from a known root filesystem each time. The layer
  cache makes an unchanged build a no-op.
- **`--cache-ttl=24h` on the base build only.** This is how Claude Code itself
  gets updated: the base rebuilds at most once a day, and the images built on
  it rebuild because their parent changed.
- **The launcher pulls Debian itself, at most once a day.** `podman image
  build` can check the registry on every build (`--pull=always` or `newer`)
  or never once the image is local (`missing`), and Podman does not store
  the date of the last check with the image. So the launcher runs
  `podman image pull` when `.pull-stamp` in its own directory is missing or
  older than 24 hours, and touches the file after a successful pull; the
  file is in `.gitignore`. The base build has no `--pull` option, so
  Podman's default `missing` applies and the build does not contact the
  registry. A new Debian image rebuilds every layer whatever the cache age,
  because the parent changed.
- **Project mounted at its host path.** Session history in the shared
  `~/.claude` volume is keyed by path, so `--continue` works across
  containers.
- **Caches are `--volume` lines in a project's `config`.** A `volume`
  instruction in a Containerfile cannot name a volume, so with `--rm` it would
  be empty on every launch. A mount point that does not exist in the image is
  created owned by root, which is why the base image pre-creates `~/.claude`
  and `~/.config` for uid 1000 and the Java base `~/.m2`; do the same for any
  new mount point in the home directory.
- **Project image untagged by default.** It is run by the id from
  `--iidfile`, so no naming scheme is needed. The cost is that
  `podman image prune` removes it and the next launch rebuilds it.
- **No JDK in the Java base.** It adds the Adoptium repository, the packages
  that `temurin-21-jdk` depends on, and Maven. A project installs the
  `temurin-<version>-jdk` it needs in its own Containerfile, as
  `claude-code-base-java/.claude-code-image/Containerfile` does; with the
  dependencies already in the base, that step adds only the JDK package.
- **Maven is pinned on `dlcdn.apache.org` and the build is meant to break.**
  dlcdn keeps only current releases, so once a newer Maven replaces the
  pinned one the step fails with a 404 and Java projects stop launching until
  `v` is bumped. That failure is the update notice: do not switch to
  `archive.apache.org` or let the cached tarball skip the request. With the
  tarball in the `maven-cache` mount the step sends a `HEAD` request for the
  same URL instead of downloading again. The check runs when the layer is
  rebuilt, which is after each rebuild of the base.

## Security model

The container is the only boundary: Claude runs with
`--dangerously-skip-permissions` and has sudo. `config` is sourced by bash on
the host, so anything that can write it gets host code execution at the next
launch. The rules below were tested with bind mounts in nested mount
namespaces and are easy to get wrong.

- The read-only mount has to be on a directory that sits directly in the
  project root. A mount on a single file, whether a read-only bind or
  `/dev/null`, is dropped by the kernel when the host replaces that file (an
  editor that saves by renaming, `git checkout`), and the file is then
  writable in the running container. The previous version masked a single
  file and had this weakness.
- Protecting a file or a subdirectory inside a writable directory does not
  work: the agent renames the parent directory and creates a new one.
- The directory is created and mounted on every launch, even when the project
  has no `config`, so the agent never gets the chance to create it.
- `.claude-code-image/` is deliberately neither created nor mounted by the
  launcher and stays writable, so the agent can maintain the Containerfile.
  Its build steps run in containers. The accepted cost is the weaker
  build-time isolation and that the shared cache mounts are writable from any
  project's Containerfile: the apt caches and `maven-cache`, whose tarball
  the Java base unpacks without a checksum.
- Whether a project image is built at all is decided by
  `claude_code_podman_image_base` in `config`, which the agent cannot change.
  The two refusal checks exist so that a Containerfile is never silently
  ignored.
- Root inside the container has no `CAP_SYS_ADMIN`. User namespaces can be
  created inside, but the inherited mounts are locked: unmounting, remounting
  read-write and re-binding the project without the submount were all refused.

## Known gaps the user has accepted

Do not fix these unprompted.

- Only the launch directory's config is protected. An agent can create
  `sub/.claude-code-container/config`, which is sourced if the launcher is
  ever run from `sub/`, and launching from a parent directory exposes the
  configs of the projects below it.
- A failed Debian pull, attempted when the last one is more than a day old, or
  a failed base build (no network, or an apt failure once the cache has
  expired) aborts the launch. There is no fallback to the existing image.
- The check for a Containerfile without a base looks only for `Containerfile`,
  not `Dockerfile`, its message does not say where to set the variable, and
  both checks run after the base build.
- Deleting and recreating `.claude-code-container/` on the host during a
  session drops the read-only mount for that session.

## Style

- Containerfiles: lowercase instructions; every `run` step is a heredoc that
  starts with a comment and `set -o errexit`; text files are written with
  `copy <<'eof' '/path'` heredocs (plus `--chmod` when the mode matters)
  rather than taken from the build context; a heredoc whose text contains an
  `eof` line uses `eof1`; apt steps carry both cache mounts; package names are
  quoted, one per line, in alphabetical order.
- Shell: long option names, single quotes for literals, `${var}` with braces,
  `[ ]` tests.
- An instruction in JSON-array form that spans lines needs `\` continuations,
  and its strings reach the program verbatim, so no shell quoting inside them.
