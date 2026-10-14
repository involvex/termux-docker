# Termux environment for Docker/Podman.

A [Termux](https://termux.dev) environment packaged into Docker image.
Environment doesn't have Android runtime components, so certain things will
not be available (DalvikVM, OpenSLES, etc...).

> **This is the `involvex` fork.** By default it builds **InVxTermux**: a
> termux-docker image with the same tooling as the
> [InVxTermux Android app](https://github.com/involvex/termux-app) — bun,
> OpenCode, Cursor Agent, `td-*` helpers, configs and widget shortcut templates.
> The tooling source of truth lives in that repo (`app/src/main/assets/invapp/`).
> Pass no `--invapp` to build a stock termux-docker image instead.

## InVxTermux image

### Build

```.sh
# from a checkout of this repo, next to a checkout of termux-app
INVAPP_SRC=../termux-app/app/src/main/assets/invapp ./generate.sh --invapp

# or, when termux-app is the current directory's sibling:
./generate.sh --invapp
```

Useful build knobs (all optional):

```.sh
TERMUX_DOCKER__IMAGE_NAME=ghcr.io/involvex/termux-docker \
TERMUX_DOCKER__BOOTSTRAP_VERSION='2026.02.12-r1%2Bapt.android-7' \
TERMUX_DOCKER__INVAPP_BUN_VERSION=1.4.2 \
TERMUX_DOCKER__INVAPP_BUN_BASELINE=0 \
TERMUX_DOCKER__INVAPP_WITH_AI=0 \
./generate.sh --invapp
```

`TERMUX_DOCKER__INVAPP_WITH_AI=1` also bakes OpenCode and Cursor Agent into
the image. They are large, so by default the image only ships their installers
and `td-ai` / `agent` fetch them on first use.

### Run

```.sh
docker run -it ghcr.io/involvex/termux-docker:aarch64
docker run -it --rm -p 5173:5173 ghcr.io/involvex/termux-docker:x86_64
```

Tags: `aarch64`, `arm`, `x86_64`, `i686`, `latest` (multiplatform manifest).
Cross-architecture runs need QEMU — see *Running ARM containers* below.

### What you get

| Feature | In the image |
|---|---|
| `bun`, `bunx` | yes on `aarch64`/`x86_64` (bun publishes no armv7/386 build — same limit as the APK); `nodejs-lts` everywhere |
| `td-dev`, `td-scaffold`, `td-clone` | yes (need bun for the first two) |
| `td-ai` + `opencode-setup`, `cursor-agent-setup`, `td-upgrade`, `td-info` | yes |
| `~/.termux/termux.properties`, hacker `colors.properties`, bash completions | yes |
| `~/.shortcuts` templates | seeded (run them by hand; no widget launcher) |
| `termux-api` CLIs | installed, but every call broadcasts an Android intent, so they fail — same as a device without the plugin APK |
| `termux-screenshot`, `td-screen-ocr` | explicit stubs explaining the Android-only requirement |
| `td-jdk-setup` / JDK | `pkg install openjdk-17`, or `td-install-extras openjdk-17` — plain `java` works in a container |
| `td-resource-linux` (proot) | intentionally absent: the container already is a Linux rootfs |

Verify with `td-info` right after the first `docker run`.

### bun is the APK's android build

The container installs the same `bun-linux-{x64,aarch64}-android.zip` bytes the
APK bundles, pinned to the checksums in `app/build.gradle`, so both sides run a
byte-identical bun. The plain Linux build cannot run here: its ELF interpreter
is `/lib64/ld-linux-x86-64.so.2`, and this rootfs is bionic (`/system` is a
symlink to `$PREFIX/opt/aosp`, whose `/system/bin/linker64` *is* present).

### Changes against upstream

Kept minimal so rebasing on `termux/termux-docker` stays easy:

| Change | Why |
|---|---|
| `--invapp` flag + `INVAPP_SRC` staging | layers the InVxTermux tooling; absent → byte-for-byte stock build |
| `TERMUX_DOCKER__IMAGE_NAME` / `TERMUX_DOCKER__BOOTSTRAP_VERSION` are overridable (`:=` instead of `=`) | lets CI and local builds retag without editing the script |
| `SYMLINKS.txt` loop skips entries that already exist | newer bootstraps ship some listed paths already in the archive; plain `ln -s` fails with `EEXIST` and, under `set -e`, kills rootfs generation |
| Seed staging strips `\r` | this fork is developed on Windows; CRLF shebangs would fail with `bad interpreter` inside the image |
| `generate.sh publish` parsed as a flag instead of `$1` | survives `--invapp publish` |
| CI publishes to `ghcr.io/involvex/termux-docker` and smoke-tests `td-info` | upstream pushes to `termux/termux-docker` on Docker Hub, which a fork must not do |

### Bun and Docker's seccomp profile

Bun uses `openat2`/`fchmodat2`. Some Docker versions' default seccomp profile
rejects them, which surfaces as `Bad system call` or an unknown syscall. If you
hit that:

```.sh
docker run -it --rm --security-opt seccomp=unconfined ghcr.io/involvex/termux-docker:aarch64
```

The container image itself is unchanged by this — it only relaxes the host
policy. `bun-doctor` prints this hint too.

## How to use

### Requirements

You should have a properly configured and running Docker or Podman
container systems. Further instructions will provide examples only for
Docker.

### Basic usage

This will start interactive login shell. Everything will look like in a
normal Termux installation.

```.sh
docker run -it termux/termux-docker
```

When using no tag, or the tag `latest`, the container will automatically
match the device architecture.

Other architectures can be installed using different tags.
Available tags:

- `aarch64`
- `arm`
- `i686`
- `x86_64`
- `latest` (multiplatform)

The other package manager for Termux, pacman, is available in a separate image:

```.sh
docker run -it termux/termux-docker-pacman
```

If architecture is not compatible with host, the additional setup will
be needed. Read this document further to learn how you can run containers
of incompatible CPU architecture.

### Running ARM containers

In order to run AArch64 container on x86(64) host, you need to setup
QEMU emulator through binfmt_misc. This can be easily done by one
command:

```.sh
docker run --rm --privileged aptman/qus -s -- -p aarch64 arm
```

Note that AArch64 and ARM containers (and in certain rare situations, some x86 containers)
sometimes work properly only in privileged mode, even on some real ARM devices.
If you want your containers to have standard privileges, a custom
seccomp profile or a custom build of Docker might be required. The custom build
of Docker limits the customizations to purely what is necessary for
the `personality()` system call, leaving the security settings of all other system
calls untouched.

Variant with privileged container:

```.sh
docker run -it --privileged termux/termux-docker:aarch64
```

Variant with seccomp unconfined profile:

```.sh
docker run -it --security-opt seccomp:unconfined termux/termux-docker:aarch64
```

Variant with custom build of Docker:

> [!NOTE]
> Example with Debian trixie and the `docker.io` package. Assumes that [`deb-src` URIs](https://wiki.debian.org/Packaging/SourcePackage?action=show&redirect=SourcePackage#With_apt-get_source) and the [`devscripts` package](https://wiki.debian.org/Packaging#Suggested_tools_to_create_an_environment_for_packaging) are already installed, and that the current user is a member of the `docker` group.

```.sh
sudo apt build-dep docker.io
apt source docker.io
cp /path/to/termux-docker/custom-docker-with-unrestricted-personality.patch docker.io-*/debian/patches/
echo 'custom-docker-with-unrestricted-personality.patch' >> docker.io-*/debian/patches/series 
cd docker.io-*/
DEB_BUILD_OPTIONS=nocheck debuild -b -uc -us
rm ../golang*
sudo apt install ../*.deb
docker run -it termux/termux-docker
```

You might then want to temporarily use `sudo apt-mark hold docker.io` to ensure the package is not automatically upgraded, causing termux-docker to stop working on the device in the future, but **not upgrading can be a security risk**. If using the patch, it is recommended to patch and recompile the Docker daemon after every upgrade.

### Non-interactive execution of commands

You can run commands in non-interactive mode. Just append them to Docker
command line.

Example:

```.sh
docker run -it --rm termux/termux-docker bash -c "apt update && apt install -yq clang"
```

### Root shell

By default root shell is disabled in container as Termux doesn't really
support usage of package manager under root account. In cases where you
really need shell with root privileges, entrypoint should be overridden.

The provided images have 2 entry points:

- `/entrypoint.sh` - the standard one which drops privileges to `system`
  user.
- `/entrypoint_root.sh` - alternate entrypoint that does not drop privileges.

Usage example:

```.sh
docker run -it --entrypoint /entrypoint_root.sh termux/termux-docker
```

## Building image

Docker:

```.sh
./generate.sh
```

Podman:

```.sh
./generate.sh --podman
```

Pacman:

```.sh
export TERMUX_PACKAGE_MANAGER=pacman
./generate.sh
```

## Known issues

There a number of known issues which may not be resolved:

* ARM containers (and in certain rare situations, some x86 containers)
  might require a custom seccomp profile or custom build of Docker to remove restrictions from the
  `personality()` system call.

* When running certain multi threaded program in 32bit containers, the PIDs can 
  balloon and easily exceed libc's limit. The only way to fix this is to set 
  `/proc/sys/kernel/pid_max` to 65535. See [termux-docker#40](https://github.com/termux/termux-docker/issues/40).
