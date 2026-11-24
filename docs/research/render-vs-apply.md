# How deploy systems split rendering from applying

Research for [brokkr#2](https://github.com/worldtree-industries/brokkr/issues/2). Feeds the grilling ticket **What brokkr outputs, and what applies it**. This doc presents options and doesn't make that decision.

Sources were read on 2026-11-24. Argo CD links point at the `latest` docs, built from `master`. Terms in **bold capitals** (Release, Deployable, Manifest, Manifest Setting, Org Default, Platform, Environment, Load Balancer, Deploy) are used as `GLOSSARY.md` defines them.

## The question in one picture

Every system here has two halves:

1. **Render**: decide what should run. This turns inputs (for brokkr: a Release, the Manifest's Manifest Settings and the Org Defaults) into a concrete desired state for one Environment.
2. **Apply**: make it run. This pushes that desired state onto a Platform and keeps it there.

The design question is what sits *between* the halves. There are three families:

- **A git repo.** The renderer commits plain desired state and an agent on the Platform pulls it. This is "GitOps" (Flux, Argo CD, the rendered-manifests pattern).
- **An OCI artifact.** The renderer pushes a bundle to a registry next to the images, and an agent pulls it (Flux OCI, Argo CD OCI sources).
- **An API call or SSH session.** The renderer, or a CLI, pushes straight to the Platform and nothing durable sits in between (Kamal, Nomad).

The family you pick decides three things brokkr cares about: what has to be installed on the Platform, where the "button" is, and whether git really holds the desired state for DR.

## Today's starting point

- `togekiss` has no GitOps agent today. The cluster repo says workloads are applied by hand with `kubectl apply -f .`, and it's "the single source of truth for the `togekiss` Kubernetes cluster" ([talos-k8s-bootstrap README](https://github.com/dfryer1193/talos-k8s-bootstrap/blob/main/README.md)).
- Harbor (`registry.werewolves.fyi`) runs *inside* `togekiss` ([README, "Container Registry"](https://github.com/dfryer1193/talos-k8s-bootstrap/blob/main/README.md); [`kubernetes/harbor`](https://github.com/dfryer1193/talos-k8s-bootstrap/tree/main/kubernetes/harbor)). This matters for DR whenever the in-between artifact lives in a registry.
- A plain Docker Compose host already exists out of band (`porygon`, which runs Sidero Omni) ([`docker-compose/`](https://github.com/dfryer1193/talos-k8s-bootstrap/tree/main/docker-compose)). It's a plausible shape for the future Docker Platform.
- Ivaldi's notes expect brokkr to "fill in a templated deploy manifest per Deployable from its `.forge.yaml` entry" and to track the latest semver tag per image ([ivaldi `docs/brokkr-notes.md`](https://github.com/worldtree-industries/ivaldi/blob/main/docs/brokkr-notes.md)).

## Flux

**What it is.** A set of Kubernetes controllers that pull desired state from a *source* (a git repo, OCI artifact or bucket) and reconcile it onto the cluster.

**Artifact between the halves.** Either a git repo or an OCI artifact.

- With git, a `GitRepository` source is fetched. A `Kustomization` then "builds the YAML manifests located at the specified `.spec.path` … validates the objects against the Kubernetes API and finally applies them on the cluster" ([Kustomization](https://fluxcd.io/flux/components/kustomize/kustomizations/)).
- With OCI, you "push the application deploy manifests, right next to the application container images", and Flux "pulls OCI artifacts from container registries, extracts the Kubernetes manifests and reconciles them" ([OCI cheatsheet](https://fluxcd.io/flux/cheatsheets/oci-artifacts/)). Flux recommends this for exactly brokkr's case: "particularly useful when the Git repository doesn't contain the final Kubernetes manifests … you can run the generators in CI and publish the resulting manifests as OCI artifacts" ([OCI cheatsheet](https://fluxcd.io/flux/cheatsheets/oci-artifacts/)). Artifacts are produced with `flux push artifact`, and the content is a tar+gzip layer with Flux media types ([OCI cheatsheet](https://fluxcd.io/flux/cheatsheets/oci-artifacts/)).
- An `OCIRepository` can pin an artifact by `.spec.ref.tag`, `.spec.ref.semver` or `.spec.ref.digest`, where "latter listed fields tak[e] precedence" ([OCIRepository, Reference](https://fluxcd.io/flux/components/source/ocirepositories/#reference)). Artifacts can be Cosign- or Notation-signed, and Flux verifies them ([OCI cheatsheet, Signing and verification](https://fluxcd.io/flux/cheatsheets/oci-artifacts/#signing-and-verification)).
- Harbor stores user-defined OCI artifacts, not just images and charts ([Harbor, Working with OCI Artifacts](https://goharbor.io/docs/main/working-with-projects/working-with-oci/)).

**What the applier needs on the Platform.** The Flux controllers running in the cluster. In Flux's own workflow example, "Flux running on the staging cluster detects the new artifact digest, pulls the manifests and applies them" ([OCI cheatsheet, Workflow examples](https://fluxcd.io/flux/cheatsheets/oci-artifacts/#workflow-examples)).

**Push-button.** Flux is pull-based and automatic by design. `.spec.interval` is required, and on each interval the controller "fetches the source … builds the Kustomization and applies it on the cluster, correcting any existing drift" ([Kustomization, Interval](https://fluxcd.io/flux/components/kustomize/kustomizations/#interval)). You can't stop it from applying what the source points at. You can only move the button:

- **The button is a git commit.** Desired state, such as the `OCIRepository` tag for an Environment, changes only when someone presses Deploy. Flux applies a change to the spec or source revision "instantly outside the interval window" ([Kustomization, Interval](https://fluxcd.io/flux/components/kustomize/kustomizations/#interval)). A `Receiver` webhook can trigger reconciliation right after the push instead of waiting for the next interval ([Receivers](https://fluxcd.io/flux/components/notification/receivers/)).
- **Suspend.** With `.spec.suspend: true`, "new Source revisions are not applied to the cluster and drift detection/correction is paused" ([Kustomization, Suspend](https://fluxcd.io/flux/components/kustomize/kustomizations/#suspend)). That's an escape hatch, not a button.
- Avoid pointing an `OCIRepository` at a moving `semver` range, which turns every tag into a Deploy. Flux's own semver example does exactly this ([OCI cheatsheet, Workflow examples](https://fluxcd.io/flux/cheatsheets/oci-artifacts/#workflow-examples)). A pinned `tag` or `digest` keeps Deploys manual.

**DR.** Git holds the Flux config itself. `flux bootstrap github` "commits the Flux manifests to the specified branch. Then it configures the target cluster to synchronize with that repository" ([flux2 `bootstrap_github.go`](https://github.com/fluxcd/flux2/blob/main/cmd/flux/bootstrap_github.go)). To rebuild, you bootstrap a new cluster against the same repo and Flux reapplies everything. If the artifacts are OCI, git holds only the *pointers* (tag or digest), so the registry must also survive. For `togekiss`, the registry is Harbor in the same cluster (see "Today's starting point").

**k8s and Docker hosts?** No. Flux applies to the Kubernetes API. The artifact *format* is generic, though: "any type of configuration can be published with the Flux CLI" ([OCI cheatsheet](https://fluxcd.io/flux/cheatsheets/oci-artifacts/)). The same push-a-bundle-to-Harbor idea could carry a Compose file, but Docker hosts would need a different puller.

## Argo CD

**What it is.** A Kubernetes-hosted GitOps controller with an API server, UI and CLI.

**Artifact between the halves.** By default it's a git repo of *unrendered* inputs, rendered inside Argo CD. The repo-server is "responsible for generating and returning the Kubernetes manifests" from repo URL, revision, path and Helm or Kustomize settings ([Architecture](https://argo-cd.readthedocs.io/en/latest/operator-manual/architecture/)). Two alternatives exist:

- **OCI images as a source**, using `repoURL: oci://…` with a `targetRevision` ([OCI](https://argo-cd.readthedocs.io/en/latest/user-guide/oci/)).
- **Source Hydrator** (beta since v3.5.0, off by default). It renders a `drySource` and pushes "hydrated manifests to git before syncing them to the cluster", writing a `manifest.yaml` into a `syncSource.path` on a target branch such as `environments/prod` ([Source Hydrator](https://argo-cd.readthedocs.io/en/latest/user-guide/source-hydrator/)). This is Argo CD's built-in version of the rendered-manifests pattern (below). It needs an extra "commit server" component and push credentials ([Source Hydrator](https://argo-cd.readthedocs.io/en/latest/user-guide/source-hydrator/)).

**What the applier needs on the Platform.** Argo CD's API server, repo server and application controller, all in a cluster. The controller "continuously monitors running applications and compares the current, live state against the desired target state" ([Architecture](https://argo-cd.readthedocs.io/en/latest/operator-manual/architecture/)).

**Push-button.** This is Argo CD's strongest fit. Automated sync is opt-in through `spec.syncPolicy.automated` ([Automated Sync Policy](https://argo-cd.readthedocs.io/en/latest/user-guide/auto_sync/)). Without it, an Application shows `OutOfSync` and waits for `argocd app sync`, which will "Sync an application to its target state" ([argocd app sync](https://argo-cd.readthedocs.io/en/latest/user-guide/commands/argocd_app_sync/)). The same operation is exposed to "Web UI, CLI, and CI/CD systems" through the API server ([Architecture](https://argo-cd.readthedocs.io/en/latest/operator-manual/architecture/)). Argo CD has its own rollback, but "Rollback cannot be performed against an application with automated sync enabled" ([Automated Sync Policy](https://argo-cd.readthedocs.io/en/latest/user-guide/auto_sync/)).

That gives two different buttons:

- **Button = sync (auto-sync off).** Git can say "vNew" while the cluster runs "vOld" until someone presses sync. That's convenient, but git then holds *what could run*, not *what runs*.
- **Button = git commit (auto-sync on).** This is the same model as Flux. Argo CD's docs point out the benefit: "CI/CD pipelines no longer need direct access to the Argo CD API server to perform the deployment. Instead, the pipeline makes a commit and push" ([Automated Sync Policy](https://argo-cd.readthedocs.io/en/latest/user-guide/auto_sync/)).

**DR.** Applications, projects and settings "can be defined declaratively using Kubernetes manifests" ([Declarative Setup](https://argo-cd.readthedocs.io/en/latest/operator-manual/declarative-setup/)), and an "app of apps" or ApplicationSet can recreate them all ([Cluster Bootstrapping](https://argo-cd.readthedocs.io/en/latest/operator-manual/cluster-bootstrapping/)). Argo CD also has `argocd admin export`/`import`, which by default includes `status` such as sync history ([Disaster Recovery](https://argo-cd.readthedocs.io/en/latest/operator-manual/disaster_recovery/)). That history lives in the cluster, not git. With auto-sync off, a rebuild from git syncs whatever git says, which may never have been deployed.

**k8s and Docker hosts?** No. Argo CD targets Kubernetes clusters.

## Rendered-manifests pattern (and Kargo)

**The pattern.** You render "before they reach the cluster and stor[e] the final, immutable YAML directly in Git or an OCI registry". Then the GitOps agent syncs "plain YAML with no abstraction layer between Git and the cluster" ([Akuity, The Rendered Manifests Pattern](https://akuity.io/blog/the-rendered-manifests-pattern)). The motivation is that with Helm or Kustomize, "Git contains the inputs to the desired state, not the desired state itself" ([Akuity](https://akuity.io/blog/the-rendered-manifests-pattern)). Per-Environment branches act as "a release bundle: a rendered snapshot of an environment's desired state. No one commits to them directly" ([Akuity](https://akuity.io/blog/the-rendered-manifests-pattern)). The article names four ways to do it (CI rendering, Argo CD sourceHydrator, Kargo and OCI artifacts) and frames the trade-off as "operational complexity versus capability" ([Akuity](https://akuity.io/blog/the-rendered-manifests-pattern)). Akuity created Argo CD and sells Kargo, so treat its rankings as a vendor's view. The pattern itself is plain.

**Kargo.** Kargo is a promotion layer, not an applier: "Promotions are different from deployments … The job of deploying … is left to a GitOps agent like Argo CD" ([Kargo Core Concepts](https://docs.kargo.io/user-guide/core-concepts/)).

- **Freight** is a bundle of artifact revisions, such as images, charts or git commits, that "must make the journey together" ([Core Concepts](https://docs.kargo.io/user-guide/core-concepts/)). That's close to a brokkr Release pinning one image per Deployable.
- A **Stage** is a promotion target that "many users equate … with environments" ([Core Concepts](https://docs.kargo.io/user-guide/core-concepts/)).
- A Stage's `promotionTemplate.steps` describe the promotion ([Working with Stages](https://docs.kargo.io/user-guide/how-to-guides/working-with-stages/)). Typically it renders, commits to a stage branch, then runs `argocd-update`, which is "commonly the last step in a promotion process" and forces an Argo CD sync ([argocd-update](https://docs.kargo.io/user-guide/reference-docs/promotion-steps/argocd-update)).
- **Push-button:** auto-promotion has to be enabled per Project through `ProjectConfig`. Otherwise you promote by hand with `kargo promote` or the UI ([Working with Stages](https://docs.kargo.io/user-guide/how-to-guides/working-with-stages/)).
- **DR:** the rendered result lands in git, so it inherits the GitOps agent's DR story.
- **k8s and Docker hosts?** No. Kargo runs in Kubernetes and integrates with Argo CD.

Kargo is roughly what brokkr itself would be on Kubernetes (Release → render → commit → trigger apply). It's useful as a reference design more than as a dependency, given v1 has one Environment and no promotion.

## Kamal

**What it is.** A CLI that deploys containers to plain servers over SSH.

**Artifact between the halves.** There is none. `config/deploy.yml` (service, image, servers, registry, env) is the input, and the CLI turns it straight into SSH and Docker commands ([Installation](https://kamal-deploy.org/docs/installation/)). `kamal deploy` logs in to the registry, builds and pushes the image (skippable with `--skip-push`), starts a new container "with the version of the app that matches the current Git version hash", switches traffic through kamal-proxy once `GET /up` returns 200, stops the old container and prunes ([kamal deploy](https://kamal-deploy.org/docs/commands/deploy/)). `--version=VERSION` targets a specific app version ([kamal deploy](https://kamal-deploy.org/docs/commands/deploy/)).

**What the applier needs on the Platform.** SSH access (root by default) and Docker, which `kamal setup` or `kamal server bootstrap` installs via get.docker.com ([Installation](https://kamal-deploy.org/docs/installation/); [kamal server](https://kamal-deploy.org/docs/commands/server/)). It also needs kamal-proxy on ports 80 and 443 ([Proxy](https://kamal-deploy.org/docs/configuration/proxy/)). No agent or control plane runs on the host. With several servers, "you need to put a load balancer in front of them" yourself ([Installation](https://kamal-deploy.org/docs/installation/)).

**Push-button.** Fully push-button: nothing happens until someone runs `kamal deploy`. Rollback is `kamal rollback <version>`, which only works if the old container still exists. "By default, old containers are pruned after 3 days" ([kamal rollback](https://kamal-deploy.org/docs/commands/rollback/)). That differs from brokkr's rule that rollback is just a Deploy of an older Release, which works for any Release still in Harbor.

**DR.** `deploy.yml` lives in git, but *which version is running* lives only on the hosts, as running containers. Nothing in Kamal's docs records deploy history anywhere else. Rebuilding means `kamal setup` against new hosts. Unless you record the version yourself, it deploys whatever you pass or check out, not "what was last deployed".

**k8s and Docker hosts?** Docker hosts only. Kamal is a model for what a push-style Docker-host applier looks like: SSH plus Docker plus a small reverse proxy that does health-checked cutover.

## Nomad

**What it is.** A scheduler with its own servers and client agents, running Docker and other task drivers.

**Artifact between the halves.** A jobspec in HCL, which "is parsed in the command line and sent to Nomad in JSON format via the HTTP API" ([Job specification](https://developer.hashicorp.com/nomad/docs/job-specification)). The handoff is an API call. `nomad job run` "submit[s] new jobs to Nomad or update[s] existing jobs" and can read the file from a local path, STDIN or a URL ([job run](https://developer.hashicorp.com/nomad/commands/job/run)).

**What the applier needs on the Platform.** Nomad servers, "expected to have either three or five servers" per region, plus client agents ([Architecture](https://developer.hashicorp.com/nomad/docs/concepts/architecture)) and a task driver such as Docker, which "handles downloading containers, mapping ports, and starting, watching, and cleaning up after containers" ([Docker driver](https://developer.hashicorp.com/nomad/docs/job-declare/task-driver/docker)). That turns a "plain Docker host" into a Nomad cluster.

**Push-button.** Native, and with a real two-step flow. `nomad job plan` is "a dry-run of the scheduler" that returns a job modify index. `nomad job run -check-index <n>` then applies "only … if the job modify index given matches the server-side version", which guards against concurrent changes ([job plan](https://developer.hashicorp.com/nomad/commands/job/plan)). Rollback is `nomad job revert <job> <version|tag>`, using Nomad's own version history ([job revert](https://developer.hashicorp.com/nomad/commands/job/revert)).

**DR.** State (jobs, nodes, allocations, ACLs) lives in the servers' Raft store and is backed up with `nomad operator snapshot save` ([snapshot save](https://developer.hashicorp.com/nomad/commands/operator/snapshot/save)). Nomad doesn't sync from git. Git holds jobspecs only if you keep them there and resubmit them yourself.

**k8s and Docker hosts?** No. Nomad replaces Kubernetes rather than targeting it. It's relevant to brokkr mainly as a model for the plan → confirm → apply button and for keeping version history at the applier.

## Comparison

| | Artifact between render and apply | Needs on the Platform | Push-button | DR from git | k8s | Docker host |
|---|---|---|---|---|---|---|
| **Flux (git)** | Git repo (raw YAML/Kustomize) | Flux controllers in cluster | Button = git commit (+ webhook to apply now) | Strong: bootstrap + repo | Yes | No |
| **Flux (OCI)** | OCI artifact in registry; git holds pointer | Flux controllers + registry access | Button = commit pinned tag/digest | Git + registry must survive | Yes | No (format is generic) |
| **Argo CD (default)** | Git repo of *inputs*, rendered in-cluster | Argo CD server, repo-server, controller | Button = `app sync` (auto-sync off) or git commit | Apps declarative; auto-sync off means git ≠ live | Yes | No |
| **Argo CD Source Hydrator** | Rendered `manifest.yaml` on env branch | Argo CD + commit server + push creds | As above | Strong: rendered state in git | Yes | No |
| **Rendered manifests (CI)** | Rendered YAML in git branch/dir (or OCI) | Any GitOps agent | Button = render + commit | Strong: exact YAML in git | Yes | Pattern yes; agent no |
| **Kargo** | Freight → rendered commit → Argo CD sync | Kargo + Argo CD in cluster | Manual unless auto-promotion enabled per Project | Via the GitOps agent | Yes | No |
| **Kamal** | None: CLI → SSH → Docker | SSH + Docker + kamal-proxy | Native (`kamal deploy`) | Config in git; running version only on hosts | No | Yes |
| **Nomad** | Jobspec → HTTP API | Nomad servers + clients + driver | Native (`plan` → `run -check-index`) | Raft snapshots; git only if you resubmit | No | Yes, as a Nomad cluster |

## Implications for brokkr

These are options and trade-offs for the grilling ticket, not decisions.

### 1. Where the button is decides whether git tells the truth

With a pull agent (Flux, or Argo CD with auto-sync), pressing Deploy means *changing git*, and git always equals what runs. With an apply trigger (Argo CD without auto-sync, Kamal, Nomad), git can drift ahead of or behind the Platform. The standing preference "desired state must be declarative in git for DR" is easiest to honour if **a Deploy is a git commit**, which brokkr makes and a pull agent applies. If brokkr instead triggers the apply, it must also commit the Deploy record so that a rebuild restores the *deployed* Release, not the latest one.

### 2. What brokkr outputs: three candidate shapes

- **(a) Rendered YAML in a git deploy repo.** This is the rendered-manifests pattern. Brokkr is the renderer and Flux or Argo CD applies. Diffs show exactly what changes, and DR is "bootstrap agent, point at repo". Brokkr needs push access to that repo.
- **(b) An OCI bundle in Harbor plus a pinned pointer in git.** Rendered output is immutable and signable, and sits next to the images. Git holds only "Environment `prod` runs bundle `mimir:vX.Y.Z@sha256:…`". DR then depends on Harbor, which today runs inside `togekiss`. That's the same dependency the images already have, but it's worth naming.
- **(c) A direct apply, with git as a record.** Brokkr talks to the Kubernetes API, or SSH/Docker later, and commits what it did. This is the simplest stack (no agent) but brokkr becomes the applier and needs cluster credentials. DR means replaying brokkr against the record, so brokkr has to be up first, which is the chicken-and-egg problem the map's "Where brokkr itself runs" lists.

### 3. Keeping the Platform seam open

None of the k8s appliers reach a Docker host, and neither Docker-host tool reaches k8s. So a single applier can't serve both. The seam has to sit *above* the applier. One layout that fits all the systems here:

- **Platform-neutral:** a Deploy record per Environment, pinning a Release (image per Deployable) plus the resolved Org Defaults and Manifest Settings. This is close to Kargo's Freight and is what git must hold for DR.
- **Platform-specific:** a renderer per Platform (k8s YAML now, a Compose file or similar later) and an applier per Platform. For k8s that's a pull agent or a direct apply. For Docker it could be a Kamal-style SSH push (health-checked cutover through a proxy) or a pull agent.

Pull-based appliers for plain Docker hosts weren't in scope and weren't researched. If a Docker Platform should be pull-based like k8s, that needs its own research ticket.

### 4. Smaller notes for the grilling

- If Flux applies, pin by tag or digest, never a `semver` range, or tagging becomes deploying ([OCIRepository, Reference](https://fluxcd.io/flux/components/source/ocirepositories/#reference)).
- Argo CD's built-in rollback is off-limits with auto-sync ([Automated Sync Policy](https://argo-cd.readthedocs.io/en/latest/user-guide/auto_sync/)). brokkr's "rollback is a Deploy of an older Release" works the same under every option, because it's just another commit or apply.
- Kamal's rollback depends on old containers that are pruned after 3 days ([kamal rollback](https://kamal-deploy.org/docs/commands/rollback/)). Brokkr's Release-based rollback avoids that limit.
- Nomad's `plan` → `run -check-index` ([job plan](https://developer.hashicorp.com/nomad/commands/job/plan)) is a good model for the button: show the diff, then apply only if nothing changed in between. A pull-based design gets the same thing from the deploy-repo diff or pull request.
- Every pull agent here (Flux, Argo CD, Kargo) is extra in-cluster software that `togekiss` doesn't run today. It would itself need to be in git and bootstrappable ([flux2 bootstrap](https://github.com/fluxcd/flux2/blob/main/cmd/flux/bootstrap_github.go); [Argo CD Declarative Setup](https://argo-cd.readthedocs.io/en/latest/operator-manual/declarative-setup/)).
