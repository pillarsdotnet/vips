# vips — backport packaging

Debian's **vips 8.18.7-1** rebuilt for Ubuntu series that ship something
older, and published to the Forgejo Debian registry.

Stonking (26.10) carries `libvips42t64 8.18.5-1`; this brings 8.18.7.

**Resolute (26.04) is deliberately not built.** vips 8.18.7 build-depends on
`libopenexr-dev (>= 3.4.6+ds-4)` and resolute ships `3.1.13-2build1` — three
major versions behind, where stonking has `3.4.6+ds-4ubuntu3`. Backporting vips
there means backporting OpenEXR first, and likely Imath behind that, for a
series whose archive libvips is only a point release behind. It has never
built: the registry has only ever held `~ubuntu26.10~` vips packages.

## What this repository contains

**Only `debian/`.** The upstream source is *not* committed — it is fetched at
build time from the `.dsc` named in `.forgejo/upstream.conf`:

```sh
UPSTREAM_DSC=https://deb.debian.org/debian/pool/main/v/vips/vips_8.18.7-1.dsc
```

That keeps the repository at a couple of hundred kilobytes instead of carrying
a source tree, and makes rebasing onto a newer upstream a one-line change.
`dpkg-source -x` verifies the tarballs against the `.dsc`'s own SHA-256 sums on
every build, so nothing is trusted on the strength of a URL alone.

The packaging delta against Debian is currently **nothing at all** — `debian/`
is Debian's `8.18.7-1` verbatim, and CI adds the backport changelog entry at
build time. This is a straight rebuild, not a patched fork. If that stops being
true, the difference will live in `debian/` and show up in `git diff`, which is
the point of having this repository at all.

**Debian removes superseded versions from its pool.** The sibling libuv1 backport hit exactly that: its first build 404'd on a
version Debian had dropped. If a
build fails to fetch, look at the pool directory above for the current `.dsc`
before assuming anything else is wrong.

## The build drops privileges

`dpkg-buildpackage` runs as an unprivileged `builder` user, not as root.
Test suites routinely refuse to run as root -- libuv's says so outright, and
that failed the sibling backport's first build.

Launchpad never hit this because its builders are unprivileged; a container is
root by default, so the build has to drop privileges itself.
`DEB_BUILD_OPTIONS=nocheck` would also make the build pass — by not running the
thing that was complaining — which is why it is not what this does.

## Versioning

`<upstream>~ubuntu<release>~ppaN`, for example
`1.52.1-4~ubuntu26.04~ppa1`. `N` is read back from what the registry already
publishes for that series, so rebuilding an unchanged upstream bumps the
revision rather than colliding. A `409` on upload means that exact version is
already there and is not treated as an error.

## Rebasing onto a newer upstream

1. Find the new `.dsc` on <https://deb.debian.org/debian/pool/main/v/vips/>.
2. Update `UPSTREAM_DSC` in `.forgejo/upstream.conf`.
3. Replace `debian/` with the new upstream's, since this backport carries no
   patches of its own: `dget -du <dsc> && dpkg-source -x *.dsc src` then copy
   `src/debian` over. Carrying an old `debian/` forward onto new source is a mismatch, not a
   rebase: Debian moved 17 files between two libuv1 revisions.
4. Push. CI rebuilds for every series in the matrix.

## Installing

The registry is an apt source inside the build containers; see
`kubuntu_bootstrap_ansible/roles/sources`. The packages are
`libvips42t64`, `libvips-dev`, `libvips-tools` and `libvips-doc`.

## Branches

- `main` — the packaging. CI builds from here.
- `github` — `main` with `.forgejo/` pruned from every commit, pushed to the
  read-only GitHub mirror. Generated; never commit to it.
