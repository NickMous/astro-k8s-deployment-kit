# astro-k8s-deployment-kit

Shared Kubernetes manifests, GitHub Actions workflows, and the nginx runtime
image that power Astro deployments. Projects consume this repo via a Git ref;
updates here flow into projects on their next ref bump.

The Symfony equivalent is
[symfony-k8s-deployment-kit](https://github.com/NickMous/symfony-k8s-deployment-kit) —
same shape, same naming, same workflow contract.

## Layout

```
base/                       # kustomize: always included
  deployment.yaml           # single web container, non-root, read-only rootfs
  service.yaml              # app Service (port 80 -> http)
  nginx-default.conf        # nginx server block (brotli/gzip static, SPA-free routing)
  kustomization.yaml

components/                 # kustomize: opt-in per project
  ssr/                      # swap the nginx pod for a Node (@astrojs/node) pod
  preview/                  # add a Node pod *next to* the nginx one (CMS live preview)
  hpa/                      # CPU HorizontalPodAutoscaler (+ removes spec.replicas)
  pdb/                      # PodDisruptionBudget

docker/
  nginx/Dockerfile          # the shared nginx + ngx_brotli runtime image

.github/workflows/
  build-nginx.yaml          # builds + publishes the shared nginx image (runs here)
  deploy-production.yaml    # REUSABLE — projects call via `uses:`
  deploy-staging.yaml       # REUSABLE — projects call via `uses:`
  cleanup-staging.yaml      # REUSABLE — projects call via `uses:`

examples/
  overlay-production/       # consume the template for production
  overlay-staging/          # consume for staging, with per-branch generator/
  overlay-ssr/              # the SSR variant
  overlay-preview/          # static site + a Node preview pod beside it
  docker/                   # project Dockerfiles (static + SSR) and .dockerignore
  workflows/                # caller stubs — copy into your project's .github/workflows/
```

## Naming convention

Generic resource names — `app`, `app-config`, `app-secret`, `app-nginx-config`,
`app-hpa`, `app-pdb`. The project's overlay sets `namePrefix:` (e.g. `mysite-`)
and kustomize rewrites every in-graph reference.

**Always include the trailing dash in `namePrefix`** — kustomize concatenates
it literally, so `namePrefix: mysite` produces `mysiteapp`.

The image placeholder is `app-image`. Overlays point it at the project's own
GHCR package, or Flux image automation rewrites the tag in the fleet repo.

## Tags this template publishes, and what to point an ImagePolicy at

A production release leaves four tags in the package:

| Tag | Deploy it? |
| --- | --- |
| `<version>` | **yes** — the multi-arch manifest |
| `production-latest` | moves, so Flux cannot detect a change from it |
| `buildstage-amd64-<version>` | no — single-arch scaffolding |
| `buildstage-arm64-<version>` | no — single-arch scaffolding |

A project that builds a second image from the same repo (see *Live preview next
to a static site*) passes its own `latest-tag`, which also keys the build cache.
Give that image a distinct `version` prefix too, and a second ImagePolicy: the
two patterns must not be able to match each other's tags.

Only `<version>` is a real release. The `buildstage-` pair exists so the deploy
job can join them into the manifest; deploying one directly gives you an image
that will not run on the other architecture's nodes.

**Anchor your `filterTags` pattern.** With `^`, the `buildstage-` prefix cannot
match, which is the point of naming them that way:

```yaml
filterTags:
  pattern: '^prod-(?P<stamp>\d{8}-\d{4})-.*$'   # ^ is load-bearing
  extract: '$stamp'
policy:
  alphabetical:
    order: asc
```

Flux uses Go RE2, which has no lookahead, so excluding a suffix like
`-(amd64|arm64)$` from the pattern is not an option — the prefix is what keeps
these apart.

## First deploy: the GHCR package

Both deploy workflows push to `ghcr.io/<owner>/<repo>` using `GITHUB_TOKEN`, and
a GHCR package carries its own access control, separate from the repository.

When the workflow pushes a package that does not exist yet, GHCR creates it,
links it to the repository and grants that repository write. Nothing to
configure — this is the normal path.

If the package **already exists** because someone pushed it by hand with a PAT,
there is no such link, and `GITHUB_TOKEN` has no standing on it. The push fails
with:

```
failed to push ghcr.io/<owner>/<repo>:<tag>: unexpected status from HEAD request
to https://ghcr.io/v2/<owner>/<repo>/blobs/sha256:…: 403 Forbidden
```

Two details make this hard to recognise:

- **The repository's Packages tab does not list it.** That tab shows linked
  packages only, and the missing link is the problem — so the package looks
  like it does not exist. Look under the account instead:
  `github.com/<owner>?tab=packages`.
- **Granting `packages: write`, in the caller or in repository settings, does
  not help.** The token has the permission; the package is refusing it. This is
  package-side ACL, not token-side.

Fix it at
`github.com/users/<owner>/packages/container/<repo>/settings` → **Manage Actions
access** → add the repository with role **Write**. Or delete the package and let
the next run recreate it, which restores the normal auto-linked state.

## Shared nginx image

`docker/nginx/Dockerfile` is the unprivileged nginx image plus the `ngx_brotli`
dynamic modules. It is built by `.github/workflows/build-nginx.yaml` and pushed
as a multi-arch image to this repo's GHCR package. Project Dockerfiles use it
as their runtime stage:

```dockerfile
FROM ghcr.io/nickmous/astro-k8s-deployment-kit:nginx-latest AS runtime
COPY --from=build /app/dist /usr/share/nginx/html
```

Pin `nginx-sha-<commit>` instead of `nginx-latest` for reproducible builds.

The server config is **not** baked into the image — `base/nginx-default.conf`
ships as a ConfigMap and is mounted at `/etc/nginx/conf.d/default.conf`. Config
fixes therefore reach every project through a `?ref=` bump, with no image
rebuild, and a config change rolls the pods (the generated name carries a
content hash).

What the config does that a default nginx doesn't:

- `brotli_static` / `gzip_static` serve the `.br` / `.gz` siblings the build
  step writes, so compression costs nothing per request.
- `try_files $uri $uri/index.html $uri/` resolves Astro's directory routes
  (`/en/sponsorship` → `.../index.html`) with a 200 instead of a redirect, so
  the served URL stays canonical.
- Cache-Control comes from a `map`, not per-location `add_header`. An
  `add_header` inside a location silently drops every header inherited from the
  server block — which is how security headers usually go missing on exactly
  the asset responses you cache hardest.
- `/_astro/` is immutable for a year, other static files a month, HTML always
  revalidates.

## Consuming the template

```yaml
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: mysite-production
namePrefix: mysite-

resources:
  - github.com/NickMous/astro-k8s-deployment-kit//base?ref=v1.0.0

components:
  - github.com/NickMous/astro-k8s-deployment-kit//components/hpa?ref=v1.0.0
  - github.com/NickMous/astro-k8s-deployment-kit//components/pdb?ref=v1.0.0
```

Each project pins its own `?ref=`, so a change here never forces every project
to upgrade at once. See `examples/overlay-production/` for a full overlay:

```bash
kubectl kustomize examples/overlay-production
```

## Required overlay pieces

### 1. `namePrefix` with a trailing dash

### 2. Traefik name references

If your ingress is a Traefik `IngressRoute`, copy `kustomizeconfig.yaml` from
the example overlay and list it under `configurations:`. Without it `namePrefix`
renames the Service to `mysite-app` while the IngressRoute keeps pointing at
`app`, and the route 404s with everything looking correct.

### 3. Ingress / certificate / middlewares

Project-specific (hostnames, TLS issuer, basic-auth). Copy and adapt from the
example overlays.

Note that `Certificate.spec.secretName` and `IngressRoute.spec.tls.secretName`
are **not** rewritten by `namePrefix` — cert-manager creates that Secret at
runtime, so it never enters the kustomize graph. The two values match literally;
rename them together.

### 4. The image

```yaml
images:
  - name: app-image
    newName: ghcr.io/nickmous/mysite
    newTag: v1.0.0
```

### 5. A pull secret, if the package is private

The base references `ghcr-login-secret`, the name fleet-infra's general-secrets
uses. Patch it in your overlay if yours differs.

That secret is **not** replicated automatically — general-secrets is a
hand-maintained list of namespaces, one Secret document each, per cluster. A new
app namespace has to be added to it, and if you forget, the failure names
nothing useful: Kubernetes silently skips an `imagePullSecrets` entry it cannot
resolve, falls back to an anonymous pull, and a private package answers `401
Unauthorized` — which reads like wrong credentials rather than absent ones. The
tell is the word "anonymous" in the kubelet message.

## Static vs SSR

The base is a static build served by nginx. For `output: 'server'` with
`@astrojs/node` in standalone mode, add the `ssr` component:

```yaml
components:
  - github.com/NickMous/astro-k8s-deployment-kit//components/ssr?ref=v1.0.0
```

It rewrites the pod to run Node: uid 1000, `envFrom` app-config + app-secret,
TCP probes instead of `/health`, and the nginx config/cache volumes removed.
Port stays 8080, so the Service, ingress and probes are unchanged.

The overlay **must** supply an `app-config` ConfigMap containing at least
`HOST=0.0.0.0` and `PORT=8080` — without `HOST` the Astro node server binds
localhost and the kubelet's probes never reach it. It is not generated by the
component because `behavior: merge` only works against generators declared in a
resource base, never one declared in a component; generating it in the overlay
also means the name hash rolls the pods when config changes. See
`examples/overlay-ssr/`.

Build the SSR image from `examples/docker/Dockerfile.ssr`, which runs as the
`node` user (uid 1000) to match.

## Live preview next to a static site

A CMS live preview needs a server: a draft has no prerendered page, so the
route rendering it must be on-demand. Moving the whole project to `ssr` for
that would cost the static delivery path — brotli/gzip of the precompressed
files, the Cache-Control map, the nginx security headers — for the sake of one
editor-facing route.

The `preview` component adds a second pod instead:

```yaml
components:
  - github.com/NickMous/astro-k8s-deployment-kit//components/preview?ref=v1.2.0
```

`app` keeps serving the static build through nginx. `preview` runs the same
project as Node from a second image, and only the CMS iframe ever reaches it.
In Astro terms the project keeps `output: 'static'` and adds an adapter, so a
build emits `dist/client` (everything prerendered, what nginx serves) plus
`dist/server` (the handful of routes carrying `export const prerender = false`).

The overlay supplies an `app-preview-config` ConfigMap (`HOST=0.0.0.0`,
`PORT=8080`), an `images:` entry for the `preview-image` placeholder, and an
ingress route to the `preview` Service — see `examples/overlay-preview/`.
Nothing about `app` changes except the project's own Dockerfile, which now
copies `dist/client` rather than `dist`.

Two consequences worth planning for:

- **Nothing sits in front of the preview pod.** Response headers the shared
  nginx config would have added have to come from an ingress middleware — as
  does whatever the CMS needs in order to frame the page (`frame-ancestors`),
  and a `X-Robots-Tag: noindex` if the preview is reachable on a public host.
- **Two images, two tags.** Give the preview image its own tag prefix and its
  own ImagePolicy; the reusable deploy workflow takes `latest-tag` and
  `environment` so a second call cannot clobber `:production-latest` or queue
  behind the first build.

## SITE_URL

Astro bakes the site URL into its output — canonical links, sitemap, RSS. That
makes it a **build**-time value, not a deploy-time one, which is the main way an
Astro pipeline differs from a Symfony one.

Both deploy workflows pass `SITE_URL` as a build arg: the production URL, or
`https://<branch-slug>.staging.example.com` for a branch deploy. Read it in
`astro.config.mjs`:

```js
export default defineConfig({
  site: process.env.SITE_URL || 'https://example.com',
});
```

The fallback keeps local `npm run build` working — an unset build arg arrives
as an empty string, which is falsy.

## Per-branch staging

`deploy-staging.yaml` renders a self-contained kustomization per branch into the
Flux fleet repo, under `apps/<app-name>/staging-branches/<slug>/`:

1. The staging overlay is copied verbatim (including `patches/`), minus
   `generator/`.
2. Every host from the `staging-domains` input is rewritten to
   `<slug>.<host>`, and any `secretName: *-staging-tls` is prefixed with the
   slug so branches don't fight over one certificate.
3. Every `*.tmpl` in `generator/` is rendered with `envsubst` and written
   without the suffix; anything else there is copied as-is. Placeholders:
   `${BRANCH_SLUG}`, `${COMMIT_SHA}`, `${STAGING_URL}`, `${IMAGE}`,
   `${APP_NAME}`.
4. The root `staging-branches/kustomization.yaml` is regenerated from whatever
   directories exist, and the result is committed and pushed (with rebase
   retries, since branches deploy concurrently).

Two ways the generated kustomization can find the base:

- **Mode A (this template is public):** its `resources:` references
  `github.com/NickMous/astro-k8s-deployment-kit//base?ref=…` and Flux fetches it
  directly. Default.
- **Mode B (template is private):** set `sync-base: true` on the workflow; it
  copies your `k8s/base/` into the fleet repo and the template references
  `../../base`.

`generator/kustomization.yaml.tmpl` ships both, one commented out.

**Two things the generated overlay must get right**, both shown in the example:

- `labels: [..., includeSelectors: true]` with a `branch: ${BRANCH_SLUG}` pair.
  Every branch ships a Deployment and Service carrying identical
  `app.kubernetes.io` labels into one namespace; without `branch` in the
  selectors, one branch's Service load-balances across every branch's pods.
- No `Namespace` resource inside a branch directory. All branch directories are
  merged into one kustomization, and two of them declaring the same Namespace is
  a duplicate-resource error. Create the staging namespace once in the fleet
  repo, outside the branch directories.

Cleanup runs from `cleanup-staging.yaml` on branch deletion. Its slug rule is
identical to the deploy workflow's on purpose — if they ever diverge, cleanup
looks for a directory that was never created under that name.

## Project workflows

The deploy logic lives here as **reusable workflows** — projects invoke them via
`uses:`, like a third-party action. Bumping the `@ref` upgrades every project.

Caller stubs live in `examples/workflows/`.

**Two gotchas:**

1. **Permissions cascade.** A reusable workflow inherits the caller's
   `GITHUB_TOKEN` permissions and can only further restrict them. If the caller
   doesn't grant `packages: write`, the run fails with `startup_failure` and no
   log output. Set them at the caller's job level.
2. **`inputs` context.** It only exists for `workflow_dispatch` /
   `workflow_call`. Referencing `${{ inputs.x }}` under a `push` or `release`
   trigger fails at startup — use `${{ github.event.inputs.x || '' }}`.

### `deploy-production.yaml`

Builds `linux/amd64` + `linux/arm64` on native runners, joins them into one
manifest, tags `:<version>` and `:production-latest`, optionally cuts a Sentry
release. It does not touch the fleet repo — Flux image automation picks up the
new tag.

| Input | Required | Description |
| --- | --- | --- |
| `version` | yes | Image tag to build (e.g. `v1.0.0`) |
| `prod-url` | no | Production URL — environment link and `SITE_URL` build arg |
| `site-url` | no | Overrides the `SITE_URL` build arg when it differs from `prod-url` |
| `dockerfile`, `build-context` | no | Default `Dockerfile` / `.` |
| `build-args` | no | Extra build args, one `KEY=value` per line |
| `latest-tag` | no | Rolling tag and build-cache key. Default `production-latest`; a second image from the same repo needs its own |
| `environment` | no | GitHub environment for the deploy job. Default `production` |
| `sentry-org`, `sentry-project`, `sentry-url` | no | Blank skips Sentry |

### `deploy-staging.yaml`

| Input | Required | Description |
| --- | --- | --- |
| `app-name` | yes | Fleet repo path `apps/<app-name>/` |
| `fleet-repo` | yes | `OWNER/repo` of the Flux fleet repo |
| `staging-domains` | yes | Comma-separated hosts to rewrite to `{slug}.<host>` |
| `staging-domain-base` | no | Branch URL base; defaults to the first `staging-domains` entry |
| `branch` | no | Override the ref the workflow ran on |
| `overlay-path` | no | Default `k8s/overlays/staging` |
| `base-path`, `sync-base` | no | Mode B: sync `k8s/base` into the fleet repo |
| `dockerfile`, `build-context`, `build-args` | no | As above |
| `platforms` | no | Default `linux/amd64` |
| `sentry-*` | no | As above |

Outputs `url` — the branch's staging URL.

### `cleanup-staging.yaml`

| Input | Required | Description |
| --- | --- | --- |
| `app-name`, `fleet-repo` | yes | As above |
| `branch-name` | yes | Branch / environment name to remove |

### Secrets in the project repo

| Secret | Required for | Notes |
| --- | --- | --- |
| `GITHUB_TOKEN` | all | Provided automatically |
| `FLEET_REPO_TOKEN` | staging + cleanup | Fine-grained PAT, `Contents: write` on the fleet repo |
| `SENTRY_AUTH_TOKEN` | optional | Only when `sentry-org` is set |
| `FLUX_WEBHOOK_URL` | optional | Activated by the repo variable `FLUX_WEBHOOK_ENABLED=true` |

`secrets: inherit` in the caller passes them all through.

## Local validation

```bash
# Render every example overlay
kubectl kustomize examples/overlay-production
kubectl kustomize examples/overlay-staging
kubectl kustomize examples/overlay-ssr
kubectl kustomize examples/overlay-preview

# Build and smoke-test the shared nginx image the way the cluster runs it:
# non-root, read-only rootfs, config from the ConfigMap.
docker build -t astro-nginx:local docker/nginx
docker run --rm -p 8080:8080 \
  --read-only --tmpfs /tmp --tmpfs /var/cache/nginx \
  --user 101:101 --cap-drop ALL \
  -v "$PWD/base/nginx-default.conf:/etc/nginx/conf.d/default.conf:ro" \
  -v "$PWD/some-astro-dist:/usr/share/nginx/html:ro" \
  astro-nginx:local

curl -sI localhost:8080/                                  # 200, no-cache
curl -sI -H 'Accept-Encoding: br' localhost:8080/_astro/…  # Content-Encoding: br
curl -s   localhost:8080/health                            # healthy
```

## Fleet-infra side

Production rollouts are Flux's job, not CI's. Per app you need a
`GitRepository` + `Kustomization` pointing at the project's
`k8s/overlays/production`, and an `ImagePolicy` + `ImageUpdateAutomation` that
bumps the tag in the `images:` block. The `name:` there must match the base
placeholder, `app-image`.

## Versioning

Tag releases (`v1.0.0`, `v1.1.0`, …); projects pin `?ref=vX.Y.Z` and
`uses: …@vX.Y.Z`. Renames and structural shifts get a major bump, additive
changes a minor one.
