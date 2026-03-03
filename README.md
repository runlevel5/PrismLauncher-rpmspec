# PrismLauncher RPM Spec for ppc64le

## Why this exists

The upstream Copr repository [g3tchoo/prismlauncher](https://copr.fedorainfracloud.org/coprs/g3tchoo/prismlauncher/) only builds for x86_64 and aarch64. There are no ppc64le builds available, so we maintain this spec file to build Prism Launcher locally on ppc64le machines running Fedora.

## What is changed from the upstream spec

1. **Java BuildRequires**: The upstream spec uses `temurin-17-jdk` (Adoptium) for Fedora > 41, which is not available for ppc64le. We replace it with `java-21-openjdk-devel`.
2. **Java source/target patch**: The bundled Java components (`libraries/launcher` and `libraries/javacheck`) set `-source 7 -target 7` for javac. JDK 21+ no longer supports source/target level 7 (minimum is 8). The patch `prismlauncher-java-source-target-8.patch` bumps these to `-source 8 -target 8`.
3. **Explicit `-p1` in `%autosetup`**: Added to ensure the patch applies correctly.

## Files

- `prismlauncher.spec` - The RPM spec file
- `prismlauncher-java-source-target-8.patch` - Patch to fix Java source/target compatibility

## Build instructions

### 1. Set up the RPM build tree

```bash
mkdir -p ~/rpmbuild/{BUILD,RPMS,SOURCES,SPECS,SRPMS}
```

### 2. Copy the spec and patch into place

```bash
cp prismlauncher.spec ~/rpmbuild/SPECS/
cp prismlauncher-java-source-target-8.patch ~/rpmbuild/SOURCES/
```

### 3. Download the source tarball

```bash
spectool -g -R ~/rpmbuild/SPECS/prismlauncher.spec
```

Or download manually:

```bash
curl -L -o ~/rpmbuild/SOURCES/PrismLauncher-10.0.5.tar.gz \
  https://github.com/PrismLauncher/PrismLauncher/releases/download/10.0.5/PrismLauncher-10.0.5.tar.gz
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
~/rpmbuild/RPMS/ppc64le/prismlauncher-10.0.5-*.ppc64le.rpm
```

Install it with:

```bash
sudo dnf install ~/rpmbuild/RPMS/ppc64le/prismlauncher-10.0.5-*.ppc64le.rpm
```

## Bumping to a new version

1. Update the `Version:` field in `prismlauncher.spec`.
2. Download the new source tarball (step 3 above).
3. Verify the patch still applies: `rpmbuild -bp ~/rpmbuild/SPECS/prismlauncher.spec`
4. If the patch fails, regenerate it against the new source.
5. Build and test.
