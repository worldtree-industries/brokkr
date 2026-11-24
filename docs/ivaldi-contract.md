# The Ivaldi contract

What Ivaldi's design expects of Brokkr. Ivaldi scaffolds Projects; Brokkr deploys their Releases. These notes moved here from `worldtree-industries/ivaldi` `docs/brokkr-notes.md`, which now points to this file.

Terms are defined in [`GLOSSARY.md`](../GLOSSARY.md). Ivaldi's own glossary defines Project, Binary, Kind and Tool.

This file records the contract as Ivaldi states it. Where Brokkr has since decided how to meet it, the decision lives in its ticket on [Map: brokkr v1](https://github.com/worldtree-industries/brokkr/issues/1) or in [`docs/adr/`](adr/), and the item below links there.

## The Manifest: `.forge.yaml`

Every Project has a Manifest, `.forge.yaml`, at its root. It is the source of truth for the Project's Binaries. Ivaldi adds entries to it, the developer may edit it, and Brokkr reads it.

```yaml
version: 1
binaries:
  api:
    kind: http
    port: 8080
    base_image: gcr.io/distroless/static-debian12:nonroot   # optional
  reaper:
    kind: job
    schedule: daily        # hourly|daily|weekly|monthly|none|<cron>
  wolfctl:
    kind: tool
```

- `binaries` maps each Binary name (the directory under `cmd/`) to its settings.
- `kind` is one of `http`, `job` or `tool`. `http` and `job` Binaries are Deployables; `tool` Binaries are Tools.
- `port` (`http` only) is unique within the Project.
- `schedule` (`job` only) is a keyword, `none`, or a standard cron expression.
- `base_image` (optional) is the image ko builds on. If it's absent, the default is `gcr.io/distroless/static-debian12:nonroot`. This is a build input, not something Brokkr acts on.
- `build: dockerfile` (optional) means the Binary is built from an unmanaged `cmd/<name>/Dockerfile` instead of by ko. Also a build input.

Which other Manifest Settings Brokkr accepts is decided in [Which Manifest Settings v1 supports](https://github.com/worldtree-industries/brokkr/issues/11).

## What Brokkr must do

- **Ignore Tools.** Binaries with `kind: tool` are never deployed.
- **Find each Deployable's image in a Release:**
  - Images live at `registry.werewolves.fyi/library/<binary>`.
  - A Release is a git tag `vX.Y.Z`. CI only builds the Deployables that changed since the previous tag, so an unchanged Deployable keeps an older semver image tag. Brokkr must not assume every image carries the newest Release's tag.
  - Pushes to `main` also publish `sha-<short>` tags. These never belong to a Release.
  - The exact rule (the most recent Release at or before this one that changed the Deployable) is in [Research: how a Release maps to one image per Deployable](https://github.com/worldtree-industries/brokkr/issues/4).
- **`http`:** run as a long-running service. Set the `PORT` environment variable from `port`.
- **`job` schedules:**
  - `none`: run once per Deploy.
  - `hourly`, `daily`, `weekly`, `monthly`: expand into a cron expression with **stable jitter**, derived from a hash of the Project and Binary names. It must be deterministic, so the schedule doesn't move between Deploys.
  - A raw cron expression is used as-is.
- **Render per Deployable:** turn each Deployable's Manifest entry, with Org Defaults, into Desired State for its Target.

## Open questions Ivaldi raised

- How Brokkr finds Projects to deploy (repo list, org scan, webhook). See [How brokkr discovers Projects and Releases](https://github.com/worldtree-industries/brokkr/issues/9).
- Which Environments exist and how promotion between them works. v1 is `prod` only; see [How Environments and their Targets are defined, and how Deployables are placed on Targets](https://github.com/worldtree-industries/brokkr/issues/18).
- The exact jitter algorithm and its hash inputs. See [How each Kind becomes Kubernetes resources](https://github.com/worldtree-industries/brokkr/issues/13).
- Whether the Manifest declares database needs that Brokkr provisions (CloudNativePG), and how migrations run. Out of scope for v1, which only references existing infrastructure; tracked in worldtree-industries/ivaldi#1.
- Probe wiring. mjolnir's router serves `/livez`, `/readyz` and `/startupz`, so an `http` Deployable's liveness, readiness and startup probes should point at those. The termination grace period must exceed the app's drain budget. Freshly generated `http` Binaries report not-ready until their readiness check is implemented.

## Boundary with the build side

Brokkr is the deploy side. Building is not its job: lint, test, and building and pushing images happen in each Project's CI, through the shared `worldtree-industries/workflows` `go.yml`. The handoff between the two sides is:

- the images in Harbor (`registry.werewolves.fyi/library/<binary>`, with semver and `sha-<short>` tags), and
- the Manifest, `.forge.yaml`, in the Project repo.
