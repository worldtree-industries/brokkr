# Research: how a Release maps to one image per Deployable

Resolves [brokkr#4](https://github.com/worldtree-industries/brokkr/issues/4), part of the [brokkr v1 map](https://github.com/worldtree-industries/brokkr/issues/1).
Terms follow brokkr's `GLOSSARY.md` (Release, Deployable, Manifest) and Ivaldi's (Project, Binary, Kind, Tool).

> **Question.** Given how the shared `go.yml` builds and tags images, how can brokkr reliably find the image for each Deployable in a Release (a Project at a `vX.Y.Z` tag, pinning the newest image built at or before that tag)?

This note recommends options. It doesn't pick one.

## Sources

All read on 2026-11-24. Line links are permalinks.

| Key | Source | Revision |
|---|---|---|
| **wf** | [`worldtree-industries/workflows`](https://github.com/worldtree-industries/workflows) | `main` = `v1.2.0` = [`1871647`](https://github.com/worldtree-industries/workflows/tree/1871647922e0739445221c34f8917ab1566e46b1) |
| **iv** | [`worldtree-industries/ivaldi`](https://github.com/worldtree-industries/ivaldi) | `main` = [`8e6953f`](https://github.com/worldtree-industries/ivaldi/tree/8e6953fcccc11bdf6c21d63af386ce5f3ab31762) |
| **mimir** | [`worldtree-industries/mimir`](https://github.com/worldtree-industries/mimir): git tags, `.forge.yaml`, `ci.yml`, Actions run logs | `main` = `fa0b897` (`v0.2.0`) |
| **k8s** | [`dfryer1193/talos-k8s-bootstrap`](https://github.com/dfryer1193/talos-k8s-bootstrap) | [`bd1f786`](https://github.com/dfryer1193/talos-k8s-bootstrap/tree/bd1f786cd7f69431bf1ac5525403b5be73ffcda8) |
| **goblog** | [`dfryer1193/goblog` `docs/research/ivaldi.md`](https://github.com/dfryer1193/goblog/blob/69802e6aa71c0e9c45fdbdcffe68872b41a04d47/docs/research/ivaldi.md#L139) | `69802e6` |
| **harbor** | Harbor v2.0 API, [`api/v2.0/swagger.yaml`](https://github.com/goharbor/harbor/blob/main/api/v2.0/swagger.yaml), and the image processor [`manifest_v2.go`](https://github.com/goharbor/harbor/blob/main/src/controller/artifact/processor/image/manifest_v2.go) | `main` |
| **oci-dist** | [OCI distribution spec](https://github.com/opencontainers/distribution-spec/blob/main/spec.md) | `main` |
| **oci-ann** | [OCI image spec, pre-defined annotation keys](https://github.com/opencontainers/image-spec/blob/main/annotations.md) | `main` |
| **ko** | [`ko build` reference](https://github.com/ko-build/ko/blob/v0.19.1/docs/reference/ko_build.md), at the `KO_VERSION` the workflow pins | `v0.19.1` |
| **gh-token** | [GitHub docs: `GITHUB_TOKEN`](https://docs.github.com/en/actions/concepts/security/github_token) | — |

Short form: `wf:.github/workflows/go.yml#L161` means that line at the revision above.

**Registry access.** `registry.werewolves.fyi` doesn't allow anonymous reads. `GET /v2/` and `GET /v2/library/mimir-server/tags/list` both return `401`. An anonymous bearer token from `/service/token` is refused for `repository:library/mimir-server:pull`, and `GET /api/v2.0/projects?name=library` returns `[]` (tested 2026-11-24). This matches k8s:[`kubernetes/harbor/README.md#L10-L19`](https://github.com/dfryer1193/talos-k8s-bootstrap/blob/bd1f786cd7f69431bf1ac5525403b5be73ffcda8/kubernetes/harbor/README.md#L10-L19): "All projects require authentication… library… Private". So nothing below comes from the registry directly. Image tags and digests come from mimir's CI logs, which record every push.

---

## 1. Which image tags exist after each event

The workflow's `plan` command picks the base to diff against, the image tag, whether to push, and the release (wf:[`internal/plan/plan.go#L97-L148`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L97-L148)). `affected` then picks the Deployables (wf:[`internal/affected/affected.go#L60-L116`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/affected.go#L60-L116)). Only those are built, each at `registry.werewolves.fyi/library/<name>:<tag>` (wf:[`.github/workflows/go.yml#L151-L163`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L151-L163)). The README's event table (wf:[`README.md#L117-L128`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/README.md#L117-L128)) agrees with the code:

| Event (Ivaldi `ci.yml` triggers on PRs to `main`, pushes to `main`, and `v*.*.*` tags) | Diffed against | Image tag pushed for each **affected** Deployable | Git tag |
|---|---|---|---|
| Push to `main` of a merge commit named `Merge pull request #N …` | highest `vX.Y.Z` **reachable** from the commit (`git tag --merged`), or nothing (every Deployable) | next `vX.Y.Z`, bumped by the PR's `semver:*` label (default patch) | `vX.Y.Z` at the merge commit, pushed **last**, only if lint, test and every image job succeeded or images were skipped |
| Same merge commit already carrying a `vX.Y.Z` tag (a re-run) | highest tag **below** it among **all** tags | `vX.Y.Z` again, overwriting the tag | none |
| Merge commit whose subject doesn't name a PR | — | none: `plan` fails | none |
| Any other push to `main` (non-merge or direct push) | the branch's previous tip (its parent after a force push) | `sha-<7 hex>` | none |
| Push of a `vX.Y.Z` tag (by hand) | highest lower `vX.Y.Z` among **all** tags, whether or not it is an ancestor | `vX.Y.Z` | none (it already exists) |
| Pull request | PR base commit | none: built but never pushed, and the job never logs in | none |

Code for each row: merge → `release()` with `latestTag` (`--merged`) (plan.go [L116-L125](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L116-L125), [L154-L197](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L154-L197), [L341-L353](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L341-L353)). Re-run → `releaseAt` then `previousTag` ([L158-L169](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L158-L169)). `previousTag` lists every tag, not just ancestors ([L428-L439](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L428-L439)). Bad subject → error ([L175-L181](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L175-L181)). Other push → `pushBase` ([L464-L473](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L464-L473)). Tag push ([L126-L136](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L126-L136)). PR never pushes: `PUSH` is false for `pull_request` (go.yml [L163](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L163)). The release tag is pushed last (go.yml [L229-L254](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L229-L254)). Mimir's triggers are at mimir:`.github/workflows/ci.yml` (`pull_request: branches: [main]`, `push: branches: [main], tags: ['v*.*.*']`).

The release job pushes the tag with `GITHUB_TOKEN`, so it doesn't start a tag-push run (go.yml [L229-L233](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L229-L233)). GitHub documents this: "events triggered by the GITHUB_TOKEN will not create a new workflow run" (gh-token). **Only tags pushed by hand trigger the tag-push row.**

### What "affected" means

A Deployable is affected (wf:`internal/affected/affected.go`) when:

- there is no base, so it's the first release ([L75-L77](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/affected.go#L75-L77), [L118-L123](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/affected.go#L118-L123));
- `go.mod` or `go.sum` changed, which affects every Deployable ([L87-L89](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/affected.go#L87-L89));
- its Manifest entry is new, or differs in any key ([L102-L105](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/affected.go#L102-L105)). A missing or invalid base Manifest counts every entry as changed ([L95-L100](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/affected.go#L95-L100));
- a changed file is a Go source file or embedded file in its package closure, or is its `cmd/<name>/Dockerfile` when it builds with `build: dockerfile` ([L106-L113](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/affected.go#L106-L113); [`packages.go#L95-L117`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/packages.go#L95-L117)).

Nothing else counts. That includes changes to `ci.yml`, the workflow pin, `KO_VERSION`, the `platforms` input, and the content behind an unpinned `base_image` tag. So an image an older Release pinned can come from an older workflow version, or be single-platform after `platforms` grows, until something else rebuilds it.

### Consequence: the invariant brokkr can rely on

A merge release diffs against the **previous release**, not the previous tip. So anything that changed a Deployable since the last release, direct `sha-` pushes included, rebuilds it under the new `vX.Y.Z`. That gives this invariant:

> For Release `vN` and Deployable `d`, the intended image is `library/d:vM`, where `vM` is the highest release in `vN`'s first-parent release chain at which `d` was affected relative to the release before it. A `sha-*` image is never the intended one.

It holds only when every release came from the merge path. Section 3 lists where it breaks.

### Real data: mimir

mimir's Manifest has one Deployable, `mimir-server` (`kind: http`), plus the Tool `mimir-generate` (mimir:`.forge.yaml`). Every release so far is a merge on workflow `v1.2.0` (mimir:`ci.yml` pins `go.yml@1871647… # v1.2.0` at all four tags). From the `go / affected` and `go / image` job logs of each `push main` run:

| Git tag | Commit | Run | `base` | `affected` | Image pushed |
|---|---|---|---|---|---|
| `v0.1.0` | `d0be8ac` | [44764469502](https://github.com/worldtree-industries/mimir/actions/runs/44764469502) | (none) | `[mimir-server]` | `library/mimir-server:v0.1.0@sha256:f52f12b3f2091ff175991677e733397fccf4e2548e0df6abd86d0ef555d0b880` |
| `v0.1.1` | `e2c9678` | [44781394092](https://github.com/worldtree-industries/mimir/actions/runs/44781394092) | `v0.1.0` | `[]` | none (`image` job skipped) |
| `v0.1.2` | `6c2f319` | [44781829055](https://github.com/worldtree-industries/mimir/actions/runs/44781829055) | `v0.1.1` | `[]` | none |
| `v0.2.0` | `fa0b897` | [44973669490](https://github.com/worldtree-industries/mimir/actions/runs/44973669490) | `v0.1.2` | `[]` | none |

So `library/mimir-server` holds exactly one tag, `v0.1.0`, while the Project's latest Release is `v0.2.0`. Each Release's `mimir-server` image is therefore `v0.1.0`. That's what prod runs today: k8s:[`kubernetes/mimir-config/deployment.yaml#L35`](https://github.com/dfryer1193/talos-k8s-bootstrap/blob/bd1f786cd7f69431bf1ac5525403b5be73ffcda8/kubernetes/mimir-config/deployment.yaml#L35) is `registry.werewolves.fyi/library/mimir-server:v0.1.0`. No direct pushes to `main` have happened (`git log --first-parent main` past `v0.1.0` is all merges), so mimir has no `sha-*` images from the current workflow. The registry couldn't be listed to check for images pushed by hand or by older workflows.

**The naive lookup "image tag == Release tag" fails for three of mimir's four Releases.**

---

## 2. Do images carry provenance today?

Yes, as image **config labels**, on both build paths, since the workflow was created (wf `de92bc0`, workflows#2):

- ko: `--image-label=org.opencontainers.image.source=<server>/<owner>/<repo>` and `--image-label=org.opencontainers.image.revision=<github.sha>` (wf:[`go.yml#L190-L207`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L190-L207)). ko's `--image-label` adds labels to the image config. Its separate `--image-annotation` adds manifest annotations, and the workflow doesn't use it (ko).
- Dockerfile: the same two keys through `build-push-action`'s `labels:` (wf:[`go.yml#L223-L225`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L223-L225)).
- mimir's `v0.1.0` run shows them applied: `SOURCE: https://github.com/worldtree-industries/mimir`, `REVISION: d0be8ac05ed6d4080b6b97bba1740416992651a0` (run 44764469502, `Build with ko` step).

What's missing:

- **No `org.opencontainers.image.version`** (oci-ann defines it as "version of the packaged software"). The tag is the only record of the version.
- **No `org.opencontainers.image.created`** from the workflow, and ko "doesn't embed timestamps in the images it produces by default" unless `SOURCE_DATE_EPOCH` is set ([ko FAQ](https://github.com/ko-build/ko/blob/v0.19.1/docs/advanced/faq.md)), which the workflow doesn't do. So ko images' `created` can't order builds. Harbor's tag `push_time` can, but a re-run resets it.
- **No manifest annotations**, so the provenance is per-platform config, not on the OCI index of a multi-platform image. Readers must open a child manifest's config.
- **No Project or Binary name**, apart from what the repository path and `source` imply.

**How to read them.** Through the OCI API: `GET /v2/<name>/manifests/<tag>` (oci-dist "Pulling manifests", end-3), then the config blob. Through Harbor: `GET /api/v2.0/projects/library/repositories/<name>/artifacts?with_tag=true` returns each artifact's `digest`, `tags[]` (with `push_time`) and `extra_attrs` (harbor swagger `Artifact`). For images, Harbor fills `extra_attrs.config` with the image config's `config` object, `Labels` included (harbor `manifest_v2.go` `AbstractMetadata`: `artifact.ExtraAttrs["config"] = config.Config`). Harbor's `q` filter covers tags (`tags=~v`, `tags=v`) and Harbor-managed labels (`labels=(id…)`), **not** OCI config labels (swagger `listArtifacts` description). Brokkr would filter by `revision` client-side. Harbor-managed "labels" are a separate feature (`/labels`, `…/artifacts/{reference}/labels`) and the workflow doesn't set any.

The OCI `tags/list` endpoint returns tags in lexical order, not semver order or push order (oci-dist "Listing Tags": "MUST be in lexical… order"; paginated with `n`/`last`). Brokkr has to sort semver itself.

---

## 3. Where the simple invariant breaks

1. **Hand-pushed tags.** A hand-pushed `vX.Y.Z` diffs against the highest lower tag among all tags, not its ancestors (`previousTag`, plan.go [L428-L439](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L428-L439)). A tag on a side branch, or one that skips ahead, can leave a Deployable unbuilt even though its code differs from the release brokkr would fall back to. Merges use `--merged` (ancestors only), so the two paths disagree.
2. **A failed release leaves orphan tags.** The `images` matrix is `fail-fast: false` (go.yml [L156-L159](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L156-L159)). If one image fails, the others are already pushed as `vX.Y.Z` but no git tag is published. The next merge computes the next version from the last **git** tag, so it reuses the same `vX.Y.Z` (`nextRelease` → `latestTag`, plan.go [L203-L230](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L203-L230)). If that merge doesn't affect a Deployable that the failed run pushed (for example because the change was reverted), the orphan `d:vX.Y.Z` stays, built from the failed run's commit, and now sits under a real Release's name. Both "tag == Release" and "newest image at or before the tag" would pick it. Only checking `revision` == the commit `vX.Y.Z` points at rejects it.
3. **Tags are mutable.** A re-run rebuilds and re-pushes the same `vX.Y.Z` (plan.go [L158-L169](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/plan/plan.go#L158-L169)). ko builds are mostly reproducible, but `base_image` is a moving tag such as `…:nonroot` ([`manifest.go#L16`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/manifest.go#L16)), and Dockerfile builds aren't reproducible. So the digest behind a tag can change. Whether Harbor's tag-immutability rules (`/projects/{p}/immutabletagrules`, harbor swagger) are on for `library` couldn't be checked without credentials. If they were, re-runs would fail to push.
4. **History before the current workflow.** Releases made by older workflows or by hand may not follow these rules or carry labels. That doesn't apply to mimir's four releases, which all ran on `v1.2.0`.
5. **`sha-*` images.** A direct push that touches `d`, followed by one that reverts it, rebuilds `d` as `sha-…` both times. The next merge then doesn't rebuild `d` (no diff from the last release). The literal "newest built at or before the tag" would be the second `sha-` image, not `d:vM`. The glossary wording should say "newest **release** image", meaning a `vX.Y.Z` tag, to exclude these.
6. **Collisions** with another Project's or a hand-built image of the same name (section 4) can put a foreign image under `library/d:vM`.

---

## 4. Image-name collisions

**The risk is real, and wider than Ivaldi Projects.** Binary names only have to match `^[a-z][a-z0-9-]{0,62}$` and be unique within one Manifest's `binaries` map (iv:[`internal/cli/binaries.go#L23`](https://github.com/worldtree-industries/ivaldi/blob/8e6953fcccc11bdf6c21d63af386ce5f3ab31762/internal/cli/binaries.go#L23); wf:[`internal/affected/manifest.go#L25`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/manifest.go#L25), [L58-L82](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/manifest.go#L58-L82)). The image path is `library/<name>` with no Project prefix (go.yml [L161](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L161)). goblog flagged this: "A binary named `server` would push `registry.werewolves.fyi/library/server` and collide with any other Project's `server`" (goblog:[`docs/research/ivaldi.md#L139`](https://github.com/dfryer1193/goblog/blob/69802e6aa71c0e9c45fdbdcffe68872b41a04d47/docs/research/ivaldi.md#L139)). `library` is also the shared project for every hand-pushed and non-Ivaldi image (k8s:[`kubernetes/harbor/README.md#L15-L19`](https://github.com/dfryer1193/talos-k8s-bootstrap/blob/bd1f786cd7f69431bf1ac5525403b5be73ffcda8/kubernetes/harbor/README.md#L15-L19), [L76-L83](https://github.com/dfryer1193/talos-k8s-bootstrap/blob/bd1f786cd7f69431bf1ac5525403b5be73ffcda8/kubernetes/harbor/README.md#L76-L83)). The cluster already pulls `library/goblog`, `golinks`, `unbound-exporter`, `werewolves.fyi` and `werewolves-dev-base` (k8s `kubernetes/*/deployment.yaml`). `werewolves.fyi` contains a dot, which Ivaldi's name rule forbids, so at least some of these come from other build paths.

**Failure mode.** Two Projects with a Deployable `server` that both release `v0.1.0` push the same `library/server:v0.1.0`. The second push silently retags it. Every Project starts at `v0.1.0` and bumps the same way, so version clashes are likely, not rare. A Deploy that resolves by tag would then run the other Project's code.

**Detection without moving anything.** Each image's `org.opencontainers.image.source` names its repository (section 2). Brokkr can refuse any image whose `source` isn't the Project's repo URL, and pin digests so a later overwrite can't change a Deploy. This detects collisions but doesn't prevent them: the overwritten Project has lost its tag.

**Cost of changing the path** to something Project-scoped, such as `library/<project>/<binary>` (Harbor and OCI allow nested repository names, oci-dist `<name>` regex) or a Harbor project per Project:

| Where | Change |
|---|---|
| **workflows** | One expression: `IMAGE` at go.yml [L161](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L161) (ko's `KO_DOCKER_REPO` and the Dockerfile `tags:` both derive from it, [L194](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L194), [L222](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L222)). The Project segment must be lowercased, because OCI names are lowercase-only (oci-dist `<name>` regex) and GitHub repo names aren't. Update the README ([L104-L108](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/README.md#L103-L108)). The image location is a contract, so this is a new major. A per-Project Harbor project also means robot-account and pull-secret work per Project. The push robot is described as having "push rights on library" (go.yml [L30](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/.github/workflows/go.yml#L30)), and the cluster pulls with `robot$togekiss-pull` (k8s harbor README [L57](https://github.com/dfryer1193/talos-k8s-bootstrap/blob/bd1f786cd7f69431bf1ac5525403b5be73ffcda8/kubernetes/harbor/README.md#L57)). Staying inside `library` avoids that. |
| **Ivaldi** | No code builds image paths. Only docs change: iv:[`README.md#L132`](https://github.com/worldtree-industries/ivaldi/blob/8e6953fcccc11bdf6c21d63af386ce5f3ab31762/README.md#L132) and [`docs/brokkr-notes.md#L36`, `#L58`](https://github.com/worldtree-industries/ivaldi/blob/8e6953fcccc11bdf6c21d63af386ce5f3ab31762/docs/brokkr-notes.md#L35-L38). Its `ci.yml` template pins the workflow SHA ([`internal/templates/ci.yml#L32`](https://github.com/worldtree-industries/ivaldi/blob/8e6953fcccc11bdf6c21d63af386ce5f3ab31762/internal/templates/ci.yml#L32)), so Ivaldi bumps the pin, and every existing Project (today mimir, plus brokkr itself) must take the new `ci.yml`. goblog notes nothing keeps Managed files current yet (goblog [§4](https://github.com/dfryer1193/goblog/blob/69802e6aa71c0e9c45fdbdcffe68872b41a04d47/docs/research/ivaldi.md#L153)). |
| **Existing images** | Unaffected Deployables aren't rebuilt (section 1), so after the switch an unchanged Deployable has **no image at the new path** until something affects it. mimir-server would sit at `library/mimir-server:v0.1.0` indefinitely. Fix with a one-off server-side copy using Harbor's "Copy artifact" (`POST /projects/{p}/repositories/{r}/artifacts?from=library/mimir-server:v0.1.0`, harbor swagger). The blobs are shared, so the digest and labels are unchanged. Alternatively brokkr falls back to the old path for releases before a cutover version. |
| **Existing deployments** | None break: old paths stay pullable, and `mimir-config/deployment.yaml#L35` keeps pointing at `library/mimir-server:v0.1.0` until brokkr owns that Deploy. The other `library/*` images aren't Ivaldi Projects and don't move. |

---

## 5. Options for resolving a Release's images

Each assumes brokkr reads the Manifest at the Release's commit to get the Deployables (non-Tool entries; iv:[`internal/kind/kind.go#L28-L30`](https://github.com/worldtree-industries/ivaldi/blob/8e6953fcccc11bdf6c21d63af386ce5f3ab31762/internal/kind/kind.go#L28-L30); wf `deployables()`, [`manifest.go#L84-L93`](https://github.com/worldtree-industries/workflows/blob/1871647922e0739445221c34f8917ab1566e46b1/internal/affected/manifest.go#L84-L93)), and that it pins each image **by digest** in the rendered output. Section 3.3 is why digest pinning matters under every option.

### A. Highest semver tag ≤ Release, from the registry

List `library/<d>` tags (OCI `tags/list` or Harbor artifacts), keep `vX.Y.Z` ≤ `vN`, take the highest, then resolve it to a digest. This is what iv:`docs/brokkr-notes.md#L37` suggests ("track the latest semver tag per image").

- **For:** simplest, needs only registry read, and gives the right answer for mimir today.
- **Against:** it trusts semver order over git ancestry (3.1). It picks up orphans (3.2) and collisions (3.6). Harbor credentials are needed anyway.

### B. A plus provenance checks (labels as guard)

As A, but accept a candidate `d:vM` only if its config labels say `source` == the Project's repo URL and `revision` == the commit git tag `vM` points at, and `vM` is an ancestor of (or equal to) `vN` (`git merge-base --is-ancestor`). On a mismatch, try the next lower tag, or fail loudly.

- **For:** rejects collisions, orphans and off-branch tags using data images already carry (section 2). Needs only one extra blob read per candidate, through Harbor's `extra_attrs.config.Labels` or the OCI config blob. No workflow change.
- **Against:** still a search. Needs both GitHub (tags, ancestry) and Harbor reads. Section 3.5's `sha-` corner case is excluded by design, so the glossary should say so.

### C. Recompute from git with the workflow's own `affected`

Walk `vN`'s release chain (`git tag --merged vN`, by semver). For each pair of consecutive releases, run `cmd/affected` (pinned to the workflows SHA the Project's `ci.yml` used at that release) to find, for each Deployable, the latest release where it was affected. Then check that the image exists and resolve its digest.

- **For:** matches what CI built, with no registry search. Also tells brokkr what changed between Releases.
- **Against:** needs a Go toolchain, `go list` and a checkout per release, and has to emulate every workflow version's logic. It's wrong wherever CI wasn't the merge path (hand tags, failed runs, pre-workflow history). It still needs B's check to be safe, so it's heavier than B without being more correct.

### D. Make CI record the mapping (workflow change)

Make the Release itself carry one image per Deployable, so lookup is exact:

- **D1, retag forward:** in the release job, give every **unaffected** Deployable's image from the previous release the new `vN` tag too. That's a registry-side tag with no rebuild: Harbor `POST …/artifacts/{reference}/tags` (harbor swagger "Create tag"), or `crane tag`. Then `library/<d>:vN` always exists and brokkr's lookup is exact. The cost is a workflow change and a one-off backfill for existing releases (mimir: tag `v0.1.0`'s digest as `v0.1.1`, `v0.1.2`, `v0.2.0`). Retagged images keep their original `revision` label, so the label means "built from", not "released as". Doing this before the release tag is pushed keeps "no tag without images".
- **D2, a release record:** the release job writes `{deployable: digest}` somewhere keyed by the tag: a GitHub Release asset, a git note, or an OCI artifact (an index or referrer, oci-dist referrers API). Brokkr reads one object. This needs a new artifact contract and a backfill.
- **D3, add a `version` label or annotation** (`org.opencontainers.image.version`, oci-ann) on `vX.Y.Z` builds, through ko `--image-annotation` / `--image-label` (ko). It doesn't remove the search, but makes B's check readable without git. Small and complementary to any option.

- **For:** turns "newest at or before" into an exact lookup that's correct by construction, and makes the image part of the Release, as the glossary intends.
- **Against:** changes the shared workflow (a release of `workflows` plus every Project's `ci.yml` pin, as in section 4) and needs a backfill. D1 adds one more push step that can fail after the images are built.

### Orthogonal: collisions

Independent of A to D: either **guard** (B's `source` check plus digest pinning, no migration) or **move** to a Project-scoped path (section 4 costs). Moving pairs naturally with D, since both change the workflow's output contract and can ship in one `workflows` major with one `ci.yml` pin bump per Project.

## Summary

- CI pushes images only for affected Deployables. The tag is `vX.Y.Z` on merges and hand-pushed tags, `sha-<short>` on other pushes to `main`, and nothing on PRs. Release tags are pushed by `GITHUB_TOKEN`, so they never trigger a tag-push build.
- Because merges diff against the previous release, a Release's image for Deployable `d` is `d:vM`, where `vM` is the latest release in its ancestry that affected `d`. `sha-*` images never qualify.
- mimir: only `mimir-server:v0.1.0` exists. Releases `v0.1.1`, `v0.1.2` and `v0.2.0` built nothing, and prod runs `v0.1.0`.
- Images already carry `org.opencontainers.image.source` and `.revision` as config labels, visible in Harbor's `extra_attrs.config.Labels`, but not `version`. The registry has no anonymous access.
- The invariant breaks on hand-pushed tags, orphans from failed release runs that reuse a version, mutable tags on re-runs, and name collisions. Checking labels against git tags guards against all of these, and digest pinning is needed under every option.
- Options: A (highest semver tag), B (A plus label checks; no workflow change), C (recompute with `affected`; heavy), D (CI retags forward or writes a release record; exact lookup, workflow change and backfill).
- Collisions are a live risk in the shared `library` project. Moving to `library/<project>/<binary>` is a one-line workflow change, but it's a contract change: a new `workflows` major, a `ci.yml` pin bump per Project, and a copy or fallback for images unaffected Deployables never rebuild.
