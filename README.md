# PrismLauncher RPM Spec for ppc64le

## Why this exists

The upstream Copr repository [g3tchoo/prismlauncher](https://copr.fedorainfracloud.org/coprs/g3tchoo/prismlauncher/) only builds for x86_64 and aarch64. There are no ppc64le builds available, so we maintain this spec file to build Prism Launcher locally on ppc64le machines running Fedora.

Having said that this RPM version also work fine with x86_64 and aarch64 and tend to be more up-to-date.

## Installing from COPR

Prebuilt packages are published to the
[`tle/PrismLauncher`](https://copr.fedorainfracloud.org/coprs/tle/PrismLauncher/)
COPR repository. Enable it and install:

```bash
sudo dnf copr enable tle/PrismLauncher
sudo dnf install prismlauncher
```

To update later, a normal `sudo dnf upgrade` picks up new builds. To remove the
repository:

```bash
sudo dnf copr remove tle/PrismLauncher
```

> **RHEL / CentOS Stream (EL) users:** the EL chroots also need EPEL and CRB
> enabled, since some dependencies are not in the base repos — see
> [Building on Fedora COPR](#building-on-fedora-copr) below. Fedora (including
> ppc64le) needs nothing extra.

## What is changed from the upstream spec

1. **Java BuildRequires**: The upstream spec uses `temurin-17-jdk` (Adoptium) for Fedora > 41, which is not available for ppc64le. We replace it with `java-21-openjdk-devel`.
2. **Java source/target patch**: The bundled Java components (`libraries/launcher` and `libraries/javacheck`) set `-source 7 -target 7` for javac. JDK 21+ no longer supports source/target level 7 (minimum is 8). The patch `prismlauncher-java-source-target-8.patch` bumps these to `-source 8 -target 8`.
3. **Explicit `-p1` in `%autosetup`**: Added to ensure the patch applies correctly.
4. **LWJGL ppc64le natives**: Prism's metadata server ships no LWJGL ppc64le
   natives, so on ppc64le modern Minecraft (26.1.2 / LWJGL 3.4.1) has no usable
   native libraries and crashes at startup. The patch
   `prismlauncher-ppc64le-lwjgl-natives.patch` injects the 11 ppc64le native
   libraries into the `org.lwjgl3` 3.4.1 component when it is parsed, so instances
   work out of the box with no manual JSON editing. The injected entries are gated
   by `rules` to the `linux-ppc64le` classifier (inert on other architectures) and
   the whole block is compiled only for `__powerpc64__` builds. Two of the entries
   point at corrected, self-contained jars (hosted on a GitHub release) because the
   official `build.lwjgl.org` ppc64le artifacts are broken:
   - the **core** jar carries the libffi `FFI_DEFAULT_ABI` fix
     ([LWJGL/lwjgl3#1126](https://github.com/LWJGL/lwjgl3/issues/1126));
   - the **openal** jar is rebuilt with GCC 14 — the official one crashes in a
     libc exit handler during shutdown (`SIGSEGV` in `__run_exit_handlers`),
     because LWJGL's CI builds OpenAL Soft 1.25.x with GCC 12, which is too old
     (it needs `std::format`, GCC 13+) and miscompiles the ppc64le exit teardown.

   The other nine modules use the official `build.lwjgl.org` ppc64le artifacts.
5. **Optional GameMode (for EL builds)**: Upstream makes Feral GameMode a hard
   CMake dependency on Linux, but `gamemode` is not packaged in EPEL/RHEL (only
   EPEL 8). The patch `prismlauncher-gamemode-optional.patch` gates it behind a
   `Launcher_ENABLE_FERAL_GAMEMODE` CMake option (default `ON`, so Fedora builds
   are unchanged). On RHEL the spec drops the `pkgconfig(gamemode)` BuildRequires
   and configures with `-DLauncher_ENABLE_FERAL_GAMEMODE=OFF`; GameMode is a
   runtime-optional feature, so the resulting build simply lacks it.

## Files

- `prismlauncher.spec` - The RPM spec file
- `prismlauncher-java-source-target-8.patch` - Patch to fix Java source/target compatibility
- `prismlauncher-no-werror.patch` - Drop `-Werror` so GCC 16 (Fedora 44+) warnings don't fail the build
- `prismlauncher-ppc64le-lwjgl-natives.patch` - Inject LWJGL ppc64le natives (core + openal from corrected jars) into the `org.lwjgl3` 3.4.1 component
- `prismlauncher-gamemode-optional.patch` - Make Feral GameMode an optional CMake feature so EL builds (no `gamemode` in EPEL) can disable it

## Building on Fedora COPR

This repo ships a [`make_srpm`](https://docs.pagure.org/copr.copr/user_documentation.html#make-srpm)
recipe at [`.copr/Makefile`](.copr/Makefile), so COPR can build the SRPM straight
from the git repo — no lookaside cache or manual tarball upload.

1. Create a COPR project and enable the `ppc64le` chroot (plus `x86_64` /
   `aarch64` if wanted).
2. Add a new package with **build method: "Make SRPM"** (a.k.a. `make_srpm`),
   pointing at this git repository. COPR runs:

   ```
   make -f .copr/Makefile srpm outdir=<resultdir> spec=prismlauncher.spec
   ```

   The Makefile installs `rpm-build`, `rpmdevtools` and `rpmautospec`, stages the
   local `*.patch` files into `SOURCES/`, downloads the upstream release tarball
   via `spectool`, and produces the SRPM. COPR then dispatches the per-chroot RPM
   builds. `%autorelease` / `%autochangelog` are resolved from this repo's git
   history at SRPM time.

To reproduce the SRPM step locally (e.g. on a Fedora box, run from the repo root):

```bash
make -f .copr/Makefile srpm outdir=/tmp/srpm
```

### RHEL / CentOS Stream / EPEL chroots need EPEL + CRB

Several build dependencies are **not** in RHEL base/AppStream and must come from
EPEL and/or the CRB (CodeReady Builder) repo, which COPR's EL chroots do not enable
by default. Symptoms (per EL version):

- EL10: `No matching package to install: 'extra-cmake-modules'` — EPEL provides
  `extra-cmake-modules`, `tomlplusplus-devel`, `scdoc`, `qrencode-devel`, …
- EL9: `No matching package to install: 'cmake(Qt6Concurrent) >= 6.2'` and
  `'cmark'` — the Qt6 `-devel` packages live in CRB; `cmark` (the binary the spec
  needs for `rhel < 10`) lives in EPEL.

This is a buildroot configuration issue, not a spec issue — it cannot be fixed from
`.spec` or `.copr/Makefile`. Fix it in the COPR project: edit each EL chroot
(`epel-9-x86_64`, `epel-10-x86_64`, …) and add to its **"Repos" / "External
repositories"** field both EPEL and CRB. The same two lines work for every EL
version — `$releasever` (9, 10, …) and `$basearch` are substituted per chroot:

```
https://dl.fedoraproject.org/pub/epel/$releasever/Everything/$basearch/
https://mirror.stream.centos.org/$releasever-stream/CRB/$basearch/os/
```

Add these per EL chroot, **not** project-wide: on a Fedora chroot `$releasever`
would be a Fedora release (e.g. 41) for which these paths don't exist, breaking the
build. Fedora chroots ship all of these in the base repos and need no extra config.

## Build instructions

### 1. Set up the RPM build tree

```bash
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
```

### 2. Copy the spec and patch into place

```bash
cp prismlauncher.spec ~/rpmbuild/SPECS/
cp *.patch ~/rpmbuild/SOURCES/
```

### 3. Download the source tarball

```bash
spectool -g -R ~/rpmbuild/SPECS/prismlauncher.spec
```

Or download manually:

```bash
curl -L -o ~/rpmbuild/SOURCES/PrismLauncher-11.0.2.tar.gz \
  https://github.com/PrismLauncher/PrismLauncher/releases/download/11.0.2/PrismLauncher-11.0.2.tar.gz
```

### 4. Install build dependencies

```bash
sudo dnf builddep -y ~/rpmbuild/SPECS/prismlauncher.spec
```

If `dnf builddep` fails on specific packages (e.g. the `temurin-17-jdk` line on non-x86_64), install them manually:

```bash
sudo dnf install -y \
  cmake ninja-build extra-cmake-modules gcc-c++ \
  java-21-openjdk-devel \
  desktop-file-utils libappstream-glib \
  libarchive-devel cmark-devel qrencode-devel scdoc \
  tomlplusplus-devel zlib-ng-compat-devel gamemode-devel \
  qt6-qtbase-devel qt6-qtnetworkauth-devel \
  qt6-qtimageformats qt6-qtsvg
```

### 5. Build the RPM

```bash
rpmbuild -ba ~/rpmbuild/SPECS/prismlauncher.spec
```

### 6. Install the built RPM

Once the build completes, the RPM will be at:

```
~/rpmbuild/RPMS/ppc64le/prismlauncher-11.0.2-*.ppc64le.rpm
```

Install it with:

```bash
sudo dnf install ~/rpmbuild/RPMS/ppc64le/prismlauncher-11.0.2-*.ppc64le.rpm
```

## Bumping to a new version

1. Update the `Version:` field in `prismlauncher.spec`.
2. Download the new source tarball (step 3 above).
3. Verify the patch still applies: `rpmbuild -bp ~/rpmbuild/SPECS/prismlauncher.spec`
4. If the patch fails, regenerate it against the new source.
5. Build and test.
