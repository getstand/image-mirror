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

The packages are **private**: getstand's administrators have disabled public
and internal packages for members. Two things follow.

- A repo that wants to pull has to be granted access on each package it uses:
  **Package settings → Manage Actions access → Add repository**, role Read.
  Without that, its `GITHUB_TOKEN` is refused (`Error response from daemon:
  denied`) even after a successful `docker login`.
- A Dockerfile `FROM` built by Railway cannot use the mirror, because Railway
  has no way to pass registry credentials to a build. Point that one image at
  `mirror.gcr.io/library/<name>:<tag>` instead, Google's mirror of Docker Hub,
  which serves the identical digest and has never refused us. The sync reads
  from it first for the same reason.

If an administrator enables public packages for the org, make the packages
public (**Package settings → Danger Zone → Change visibility**) and both
caveats go away, including for Railway.

## Setting up a new repo

In order:

1. **Check the tag is mirrored.** Every tag you pull must be a line in
   `images.txt`; a tag that is not there answers `manifest unknown`. If yours
   is missing, add it (see "Adding an image") and wait for that sync to go
   green before changing your workflow.
2. **Grant the repo on each package it uses.** For each of `postgres`, `redis`,
   `ruby`, `moby/buildkit`, ... open the package from
   https://github.com/orgs/getstand/packages?repo_name=image-mirror, then
   **Package settings → Manage Actions access → Add repository**, role Read.
   This needs admin on the package, which comes with admin on this repo; ask
   @davyaginuma if you do not have it.
3. **Keep `packages: read` on the job's token.** A workflow with no
   `permissions:` block has it by default. A workflow that declares
   `permissions:` for any reason drops every scope it does not list, and the
   pull then fails with the same `denied` as a missing grant, so add
   `packages: read` to that block.
4. **Change the workflow.** A service container takes the `credentials:` block
   under "Using it". A `docker build` in CI, whether for the buildx driver
   image or for a Dockerfile `FROM`, needs a login step first:

   ```yaml
   - uses: docker/login-action@v4
     with:
       registry: ghcr.io
       username: ${{ github.actor }}
       password: ${{ secrets.GITHUB_TOKEN }}

   - uses: docker/setup-buildx-action@v4
     with:
       driver-opts: image=ghcr.io/getstand/mirror/moby/buildkit:buildx-stable-1
   ```

   A Dockerfile that Railway also builds keeps its `FROM` on `mirror.gcr.io`
   (see "Using it"); the login only helps builds that run on GitHub.
5. **Check the first run.** The job log for a service container shows
   `Downloaded newer image for ghcr.io/getstand/mirror/<name>:<tag>` and a
   buildx setup shows the mirror's buildkit image being pulled. `denied` after
   a successful login means step 2 or 3; `manifest unknown` means step 1.

A run that Dependabot triggers uses a read-only token and the actor
`dependabot[bot]`. That should still pull a package the repo is granted on,
but no run has exercised it yet; if the first one is refused, that is where to
look.

## Adding an image

Add a line to `images.txt`, written the way you would name the image on Docker
Hub, with an explicit tag:

```
postgres:14.18
moby/buildkit:buildx-stable-1
```

Merging to `main` runs the sync, and the image is pullable a few minutes later.

A new package starts private and is linked to this repo, so only this repo's
workflows can pull it. Grant each consuming repo access as described under
"Using it".
An image that does not live on Docker Hub takes its full upstream reference as
a second column (see the comments in `images.txt`).

Removing a line stops refreshing the package; it does not delete it. Delete
unused packages from the org's package settings.

## How the sync works

The sync is the GitHub Actions workflow in `.github/workflows/sync.yml`. It
runs on three triggers:

- **Mondays at 06:17 UTC.** This is the weekly refresh.
- **A push to `main`** that touches `images.txt`, `bin/sync` or the workflow,
  so adding an image syncs it straight away.
- **On demand**: Actions → Sync → Run workflow.

### What a run does

The job downloads a pinned `crane` release and checks its sha256 before using
it, logs in to `ghcr.io` with the job's own `GITHUB_TOKEN` (which has
`packages: write` on this repo's packages), and runs `bin/sync`. For every line
of `images.txt` the script:

1. Reads the upstream digest, trying `mirror.gcr.io` first, then
   `public.ecr.aws`, then Docker Hub, and taking the first that answers. The
   refresh therefore does not depend on any one registry's per-IP allowance.
2. Compares it with the digest the mirror serves. If they match it prints
   "up to date" and moves on; nothing is pulled or pushed.
3. Otherwise copies the image with `crane copy`, registry to registry with no
   Docker daemon, including every platform in the index, then reads the digest
   back from the mirror and checks it equals the upstream one.
4. Appends `<mirrored ref>@<digest>` to a new lockfile.

If any image fails every source, or its copy fails, the run exits non-zero and
the old `images.lock` is left alone. Otherwise the new lockfile replaces it and
the last step commits it as `github-actions[bot]`. A run where nothing changed
commits nothing.

### What to expect over time

- **Floating tags drift, pinned tags do not.** `postgres:latest` and
  `redis:latest` move whenever Docker Hub publishes a new image, so most
  Mondays copy those. A tag like `ruby:3.3.12-slim-bookworm` moves only when
  its maintainers rebuild that exact tag, usually for a base-OS security patch.
  `git log images.lock` is the record of when the mirror changed and to what,
  and answers what `postgres:latest` resolved to on a given date.
- **The mirror lags upstream by up to a week.** A consumer pulling an unpinned
  tag gets what the last sync copied, not Docker Hub's current image, and the
  tag changes only when a sync runs, never between two CI runs. A Dockerfile
  base image should be pinned by digest so a Monday refresh can never change
  what production builds from without a reviewed PR (Dependabot's `docker`
  ecosystem bumps the digest).
- **The bot's commit keeps the schedule alive.** GitHub disables scheduled
  workflows on a public repo after 60 days without a commit. The `images.lock`
  commits count, and the floating tags mean there is nearly always one. If
  every tag in the list were ever pinned and stable, the schedule would need
  watching; if it is disabled, re-enable it from the Actions tab. A disabled
  schedule means the mirror stops refreshing, not that it stops serving.

### When a run fails

The workflow goes red on the Actions tab and GitHub emails whoever last
committed to the workflow. Nothing posts to Slack. The mirror keeps serving
its last good copy of every image, so consuming CI does not break; what breaks
is only that the mirror stops tracking upstream until the next green run.

## Caveats

- `crane copy` of an unpinned tag copies every platform in the index. Storage
  for the org's packages has not been a concern, so nothing here trims it.
