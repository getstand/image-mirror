# image-mirror

A copy of every container image our CI jobs and Dockerfiles pull, served from
`ghcr.io/getstand/mirror/<name>:<tag>`. Pull from here instead of Docker Hub or
`public.ecr.aws`.

## Why

GitHub's runners and Railway's builders share IP addresses with every other
tenant, and public registries ration anonymous pulls per IP. Docker Hub's
hourly limit failed socotra_webhooks CI on 2026-10-09 (PROD-8842); the
`public.ecr.aws` mirror it moved to has a non-adjustable 500 GB per month cap
that ran out on 2026-10-16 (PROD-9081). A registry we own is the only one that
cannot ration us. GHCR is free for the org, and every Actions job already holds
a token for it.

## Using it

A service container in a workflow:

```yaml
services:
  postgres:
    image: ghcr.io/getstand/mirror/postgres:14.18
    credentials:
      username: ${{ github.actor }}
      password: ${{ secrets.GITHUB_TOKEN }}
```

A Dockerfile:

```dockerfile
FROM ghcr.io/getstand/mirror/ruby:3.3.12-slim-bookworm
```

A `docker build` or `docker pull` in a workflow step, after
`docker/login-action` against `ghcr.io` with the same token.

The packages are public (they are byte-for-byte copies of public images), so
Railway and other builders that cannot pass registry credentials to a
Dockerfile `FROM` can pull them too. Keep the `credentials:` block anyway:
GitHub documents no pull limit for packages, but authenticated pulls are the
case its docs describe.

## Adding an image

Add a line to `images.txt`, written the way you would name the image on Docker
Hub, with an explicit tag:

```
postgres:14.18
moby/buildkit:buildx-stable-1
```

Merging to `main` runs the sync, and the image is pullable a few minutes later.

**A new package starts private**, whatever the repo's visibility, and GitHub
offers no API to change that. After the first sync of a new image, open the
package in the [org's packages](https://github.com/orgs/getstand/packages),
then **Package settings → Danger Zone → Change visibility → Public**. Until
then only this repo's own workflows can pull it: another repo's
`GITHUB_TOKEN` is refused, and so is Railway.
An image that does not live on Docker Hub takes its full upstream reference as
a second column (see the comments in `images.txt`).

Removing a line stops refreshing the package; it does not delete it. Delete
unused packages from the org's package settings.

## How the sync works

`bin/sync` runs every Monday, on every change to `images.txt`, and on demand
(**Actions → Sync → Run workflow**). For each image it reads the upstream
digest, skips the copy when the mirror already serves it, and otherwise copies
the whole manifest list with `crane copy`, so the mirrored digest equals the
upstream digest on every platform.

Docker Hub images are read through `mirror.gcr.io` first, then
`public.ecr.aws`, then Docker Hub, so the weekly refresh itself does not depend
on any one registry's per-IP allowance.

`images.lock` records the digest behind every tag the mirror serves, committed
by the sync whenever one changes. That is also what `git log` answers when
someone asks what `postgres:latest` resolved to on a given date: an unpinned
tag here changes only when a sync runs, never between two CI runs.

## Caveats

- A scheduled workflow on a public repo is disabled after 60 days without a
  commit. The `images.lock` commits count, and `postgres:latest` moves often
  enough that this has not been a problem; if the schedule is ever disabled,
  re-enable it from the Actions tab. A disabled schedule only means the mirror
  stops refreshing, not that it stops serving.
- `crane copy` of an unpinned tag copies every platform in the index. Storage
  for public packages is free, so nothing here trims it.
