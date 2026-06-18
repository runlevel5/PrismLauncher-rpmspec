# COPR "make_srpm" build method.
#
# COPR invokes:  make -f .copr/Makefile srpm outdir=<resultdir> spec=<specpath>
# (cwd is the repo root). We download the upstream release tarball referenced by
# Source0, stage the local patches alongside it, then build the SRPM. No
# lookaside cache and no manual tarball upload are involved.

# Note: COPR invokes this Makefile with `spec=<path>` on the command line, which
# would override a lowercase `spec :=` here, so we use an uppercase SPEC it does
# not touch.
SPEC    := prismlauncher.spec
topdir  := $(CURDIR)/_build

srpm:
	# rpmdevtools provides spectool. The spec uses a static Release + changelog
	# (no rpmautospec), so SRPM builds are self-contained and deterministic.
	dnf -y install rpm-build rpmdevtools
	mkdir -p $(topdir)/SOURCES
	# The Patch entries are local files (no URL), so spectool will not fetch
	# them. Stage them next to the about-to-be-downloaded tarball.
	cp *.patch $(topdir)/SOURCES/
	# Download Source0 (the upstream PrismLauncher GitHub release tarball).
	spectool -g -R --define "_topdir $(topdir)" $(SPEC)
	rpmbuild -bs --define "_topdir $(topdir)" $(SPEC)
	cp $(topdir)/SRPMS/*.src.rpm $(outdir)

.PHONY: srpm
