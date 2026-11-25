---
status: accepted
---

# Harbor, verified against git, decides which Releases exist

A Deployable's Releases are the `vX.Y.Z` tags on its image in Harbor, `registry.werewolves.fyi/<project>/<binary>`. A tag counts only if:

- the image's `org.opencontainers.image.source` label names the Project's repo;
- a git tag of the same name exists;
- the image's `org.opencontainers.image.revision` label is that git tag's commit.

The Control Plane looks this up on demand, when someone asks. It reads the Deployable's Manifest entry from GitHub at that tag and pins the image by digest at Deploy time. It works because CI rebuilds a Deployable whenever its code or its Manifest entry changes, so a git tag that didn't touch a Deployable has nothing new to deploy for it. Decided in [Map: brokkr v1 → How brokkr discovers Projects and Releases](https://github.com/worldtree-industries/brokkr/issues/9).

## Considered Options

- **Git tags define Releases; Harbor only supplies images.** A Release names the whole Project, and each Deployable resolves to the image from the newest earlier tag that changed it. Rejected: a Deployable is only ready to ship once its image is in Harbor, and naming `v0.2.0` would quietly ship `server v0.1.0`, which misleads whoever pressed Deploy.
- **Match the image tag to the Release tag and stop there.** Rejected: hand-pushed tags, orphan images from failed release runs, tags overwritten on a re-run, and name collisions would all get through unnoticed.
- **CI records the Release-to-image mapping** (retag forward, a release record, or a `version` label). Not needed now: it means a workflow change and a backfill. It's the fallback if the label checks prove brittle.
- **Webhooks or polling keep a catalogue current.** Rejected: Deploys are push-button, so nothing needs to know about a Release before someone asks for it. GitHub doesn't retry failed deliveries, so an API pass would be needed anyway.

## Consequences

- `--release vX.Y.Z` is an exact match. If any named Deployable has no such Release, the whole Deploy is refused. `latest` means each Deployable's own newest Release.
- A Project is any org repo with at least one verified Release image. There's no opt-in list.
- Images move from `library/<binary>` to one Harbor project per Project, created by Ivaldi. Existing Release images are copied across with their digests kept, so Brokkr knows only the new path.
- Brokkr needs read access to both GitHub (an org App with Contents and Metadata read) and Harbor (a read-only robot).
