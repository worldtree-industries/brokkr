# How teams without Kubernetes deploy containerized code

Research for [brokkr#16](https://github.com/worldtree-industries/brokkr/issues/16). It feeds the non-Kubernetes Platform seam on the [v1 map](https://github.com/worldtree-industries/brokkr/issues/1), and it fills the gap that [render-vs-apply](https://github.com/worldtree-industries/brokkr/blob/research/render-vs-apply/docs/research/render-vs-apply.md) left open: "Pull-based appliers for plain Docker hosts weren't in scope". This doc presents options and doesn't pick one.

Sources were read on 2026-11-24. GitHub star counts come from the GitHub API on that date. **Bold capitals** (Release, Deployable, Platform, Environment, Deploy, Desired State, Control Plane, Agent) are used as `GLOSSARY.md` defines them. **Target** is a proposed term used here for one running instance of a Platform, such as the `togekiss` cluster or a single Docker host.

Kamal and Nomad are already covered in render-vs-apply. This doc only adds what that note didn't cover: drift, secrets, and fit against #16's constraints.

## Summary

- **A plain Docker host has no reconciler.** On Kubernetes, `kubectl apply` writes desired state and controllers keep enforcing it. On a Docker host, `docker compose up` is one-shot: it brings containers in line with the file once, then exits. Only the daemon's restart policy keeps watching, and it only restarts containers that stopped. Something has to play the controller's role, whether that's a GitOps agent, systemd, Swarm managers or Nomad.
- **Compose is the dominant format by a wide margin.** It's the native format of every Docker-host GitOps tool checked here (Doco-CD, Komodo, Portainer) and of Swarm stacks. Stack Overflow 2026 reports "Docker / Docker Compose" at 59.1% against Kubernetes at 26.0%. Swarm, Nomad, Kamal and the GitOps agents don't appear in its list at all.
- **Doco-CD fits the constraints most closely among the Compose agents.** It runs on the host, pulls Compose from git or OCI, resolves `op://` 1Password references when it deploys, and can redeploy when containers go missing or unhealthy. It's also the least adopted (1.8k stars).
- **Komodo and Portainer are fuller platforms with a central server.** Portainer's "Force redeployment" overwrites local drift on every poll. Komodo stores secrets in its own database or config, which breaks "secrets are referenced, never held" unless you add something else.
- **Podman Quadlet makes systemd the reconciler.** Desired State would be `.container` unit files, or Kubernetes YAML through `.kube` units and `podman kube play`. There's a git-pull agent (FetchIt), but it's small (178 stars), and its README says new features are unreleased.
- **Docker Swarm has built-in reconciliation**, and a one-node swarm is legitimate. However, `docker stack deploy` only accepts the legacy Compose v3 format, and Swarm secrets are stored in Swarm's Raft log, so Swarm holds them.
- **k3s turns the box into a Kubernetes Target.** A single-node k3s on the local-LLM box would make the non-Kubernetes Platform unnecessary for that box. The trade-off is running Kubernetes on it.
- **Referenced secrets can stay rebuildable with stock tools.** Put `op://` references in the Desired State and resolve them on the Target at apply time. A by-hand rebuild is `op run -- docker compose up -d` (1Password CLI plus Docker), with no brokkr involved.
- **Push-button needs auto-update off everywhere.** Komodo `auto_update`, Portainer "Re-pull image", Podman `AutoUpdate=` and Watchtower all redeploy whenever a new image appears. Pin images by digest and leave these off, so a git commit by the Control Plane is the only trigger.

## Background: three layers on a non-Kubernetes host

If you know Kubernetes, it helps to split a Docker host into the three jobs Kubernetes does for you.

| Job | Kubernetes | Plain Docker host |
|---|---|---|
| **Spec format** (what to run) | YAML objects | A Compose file, Quadlet units, a Swarm stack, a Nomad jobspec, or Kamal's `deploy.yml` |
| **Apply** (make it so, once) | `kubectl apply` | `docker compose up -d`, `systemctl daemon-reload && systemctl restart`, `docker stack deploy`, `nomad job run`, `kamal deploy` |
| **Reconcile** (keep it so) | Controllers, always on | Nothing by default, except the daemon's restart policy |

`docker compose up` "picks up the changes by stopping and recreating the containers" when a service's config or image changed. It recreates only what changed, much like `kubectl apply` ([docker compose up](https://docs.docker.com/reference/cli/docker/compose/up/)). With `-d` it leaves containers running and exits, and `--wait` makes it block until services are "running or healthy" ([docker compose up](https://docs.docker.com/reference/cli/docker/compose/up/)). Orphans, meaning services you deleted from the file, are only removed with `--remove-orphans` ([docker compose up](https://docs.docker.com/reference/cli/docker/compose/up/)), which is the Compose version of `kubectl apply --prune`.

After that, the Docker daemon only applies the restart policy. `unless-stopped` restarts a container "if it exits or if the daemon restarts, but not if you stopped the container yourself" ([Start containers automatically](https://docs.docker.com/engine/containers/start-containers-automatically/)). It won't recreate a container someone deleted, revert a `docker run` someone typed by hand, or notice that the file in git changed. That's the gap brokkr's Agent would fill.

The tools below differ mainly in **who reconciles** and **which format they read**.

## Adoption (indicative)

None of these sources measures "deploys to own hosts without Kubernetes" directly. Treat the figures as rough signals. The surveys count developers who used a tool, not production deployments. Stars measure interest, not use.

| Tool | Survey signal | GitHub stars (2026-11-24) | Notes |
|---|---|---|---|
| Docker / Docker Compose | 59.1% of 10,889 respondents ([SO 2026, Cloud development](https://survey.stackoverflow.co/2026/technology/data)); "Docker" 71.1% of 24,473 in 2025 ([SO 2025](https://survey.stackoverflow.co/2025/technology#1-cloud-development)) | [docker/compose](https://github.com/docker/compose) 38,985 | SO 2026 merged "Docker / Docker Compose" into one option |
| Kubernetes (comparison) | 26.0% ([SO 2026](https://survey.stackoverflow.co/2026/technology/data)); 82% of *container users* run it in production ([CNCF 2025 survey](https://www.cncf.io/announcements/2026/01/20/kubernetes-established-as-the-de-facto-operating-system-for-ai-as-production-use-hits-82-in-2025-cncf-annual-cloud-native-survey/)) | [k3s-io/k3s](https://github.com/k3s-io/k3s) 34,507 | CNCF respondents skew cloud-native |
| Podman (incl. Quadlet) | 13.0% ([SO 2026](https://survey.stackoverflow.co/2026/technology/data)) | [podman](https://github.com/podman-container-tools/podman) 33,382 | No separate figure for Quadlet |
| Portainer | Not listed | [portainer/portainer](https://github.com/portainer/portainer) 38,899 | |
| Komodo | Not listed | [moghtech/komodo](https://github.com/moghtech/komodo) 13,014 | GPL-3.0 |
| Doco-CD | Not listed | [kimdre/doco-cd](https://github.com/kimdre/doco-cd) 1,812 | Apache-2.0, pushed 2026-11-23 |
| Kamal | Not listed | [basecamp/kamal](https://github.com/basecamp/kamal) 14,754 | |
| Nomad | Not listed | [hashicorp/nomad](https://github.com/hashicorp/nomad) 17,120 | BUSL 1.1 since 1.7.0 ([LICENSE](https://github.com/hashicorp/nomad/blob/main/LICENSE)) |
| Docker Swarm | Not listed | [moby/swarmkit](https://github.com/moby/swarmkit) 3,659 | Mirantis commits to support Swarm in MKE 3 "through 2030" ([Mirantis, 2025-07-01](https://www.mirantis.com/blog/mirantis-guarantees-long-term-support-for-swarm/)) |
| FetchIt (Podman GitOps) | Not listed | [containers/fetchit](https://github.com/containers/fetchit) 178 | AGPL-3.0 |
| Self-hosted PaaS | Not listed | [Coolify](https://github.com/coollabsio/coolify) 66,419; [Dokploy](https://github.com/Dokploy/dokploy) 38,677; [Dokku](https://github.com/dokku/dokku) 32,235; [CapRover](https://github.com/caprover/caprover) 15,210 | See "Other approaches" |
| Watchtower | Not listed | [containrrr/watchtower](https://github.com/containrrr/watchtower) 24,578, **archived** | "This project is no longer maintained" ([README](https://github.com/containrrr/watchtower/blob/main/README.md)) |

What this shows: Compose is the default way to describe containers on a host, and Kubernetes leads among orchestrators. Every other tool is a niche, with self-hosted PaaS products drawing the most stars.

## Docker Compose, bare

**Format.** A Compose file. `image` accepts `[<registry>/][<project>/]<image>[:<tag>|@<digest>]`, so a Release's images can be pinned by digest ([Compose services: image](https://docs.docker.com/reference/compose-file/container-spec/)). GPUs are requested in the file through `deploy.resources.reservations.devices` with `capabilities: [gpu]` ([GPU support](https://docs.docker.com/compose/how-tos/gpu-support/)), which matters for a local-LLM box.

**Apply.** `docker compose up -d --remove-orphans --wait` (see "Background").

**Drift.** None beyond the restart policy. Teams that run bare Compose usually rerun `up` by hand, from cron or from CI over SSH.

**Secrets.** Compose secrets come from a `file` or from an `environment` variable on the host, and are mounted at `/run/secrets/<name>` ([Compose secrets reference](https://docs.docker.com/reference/compose-file/secrets/); [Use secrets](https://docs.docker.com/compose/how-tos/use-secrets/)). Compose has no secret-manager integration of its own. 1Password closes the gap from outside: `op run` loads environment variables whose values are `op://` secret references and "runs the provided command in a subprocess with the secrets made available as environment variables only for the duration of the process" ([Load secrets into the environment](https://developer.1password.com/docs/cli/secrets-environment-variables/)). `op inject` renders a template file the same way ([Load secrets into config files](https://developer.1password.com/docs/cli/secrets-config-files/)). Together they give a by-hand recipe that keeps only references in git:

- git holds `compose.yaml` with a `secrets: db: { environment: DB_PASSWORD }` entry, plus an env file with `DB_PASSWORD="op://prod/mimir/db-password"`;
- rebuild is `op run --env-file=prod.env -- docker compose up -d`.

Machines authenticate with a 1Password service account token scoped to specific vaults ([Service Accounts](https://developer.1password.com/docs/service-accounts/)). The token is the one secret the Target must hold.

**Fit.** The format and by-hand rebuild fit well. Drift correction is missing, and so is a trigger, unless something runs `up` for you.

## Compose with a pull-style GitOps agent

Three tools put an always-on process next to Compose. They differ in where that process lives and what it stores.

### Doco-CD

**What it is.** A single container on the Docker host that "automatically deploys and updates Docker Compose projects/services and Swarm stacks using polling and webhooks" ([README](https://github.com/kimdre/doco-cd/blob/main/README.md)). Its own description is "a simple Portainer or ArgoCD alternative for Docker" ([README](https://github.com/kimdre/doco-cd/blob/main/README.md)).

**Source.** A git repository or an OCI artifact ([README](https://github.com/kimdre/doco-cd/blob/main/README.md)). Per-deployment settings live in a `.doco-cd.yml` next to the Compose files ([Deploy Settings](https://doco.cd/latest/Deploy-Settings/)). The Compose file itself stays stock.

**Trigger.** Polling, every 3 minutes by default, with no inbound connection needed. A webhook from the git provider is the alternative ([Getting Started](https://doco.cd/latest/Getting-Started/)). A REST API can trigger polls and control projects ([REST API](https://doco.cd/latest/Endpoints/REST-API/)). There are also timezone-aware sync windows that limit when deployments may happen ([README](https://github.com/kimdre/doco-cd/blob/main/README.md)).

**Drift.** On a change in git, it pulls, resolves secrets and deploys the Compose services ([Core Concepts](https://doco.cd/latest/Core-Concepts/)). `remove_orphans` defaults to `true` ([Deploy Settings](https://doco.cd/latest/Deploy-Settings/)). Optional **reconciliation** "compares the currently running Docker services/containers with the expected deployment state". On configured container events (default `unhealthy`), it reapplies the deployment or restarts the container, and a redeploy "restores missing containers without forcing recreation of unchanged services" ([Deploy Settings, Reconciliation](https://doco.cd/latest/Deploy-Settings/)). One caveat: "The currently implemented state will be lost when doco-cd gets restarted and will be restored in the next poll or webhook event" ([Deploy Settings, Reconciliation](https://doco.cd/latest/Deploy-Settings/)). This is event-driven, not a Kubernetes-style continuous diff of every field.

**Secrets.** It supports external secret providers including 1Password (service account) and 1Password Connect. A `.doco-cd.yml` maps a variable to a reference such as `DB_PASSWORD: "op://vault/item/field"` ([1Password provider](https://doco.cd/latest/External-Secrets/1Password/)). Resolved values are usable "as Compose secrets and configs" or through interpolation ([External Secrets](https://doco.cd/latest/External-Secrets/)). SOPS encryption is also supported ([README](https://github.com/kimdre/doco-cd/blob/main/README.md)).

**Fit.** Git is the source, the agent runs on the Target, secrets are references, and the Compose file can still be applied by hand. Doco-CD's own config (`.doco-cd.yml`, `external_secrets`) is not stock. A by-hand rebuild would have to redo its secret mapping with `op run`. Adoption is the smallest of the three.

### Komodo

**What it is.** "A web application for managing servers, builds, deployments, and automated procedures." It has two parts: **Core** ("Web server hosting the API and browser UI") and **Periphery**, a "small, stateless agent running on each connected server. Called by Core to perform actions" ([What is Komodo?](https://komo.do/docs/intro)). Periphery is the agent on the host, but it's driven by Core.

**Source.** A **Stack** is a Compose project whose files can be written in the UI, kept on the host, or cloned from a git repo, with "webhooks to auto-redeploy on push" ([Compose stacks](https://komo.do/docs/deploy/compose)). **Resource Syncs** declare Komodo resources (servers, stacks and so on) in TOML. Core polls them and "alert[s] about pending changes", and "the UI will display the computed sync actions and only execute them upon manual confirmation", unless a git webhook is set to execute them on push ([Sync Resources](https://komo.do/docs/automate/sync-resources)). "Managed Mode" writes UI edits back to git as commits ([Sync Resources](https://komo.do/docs/automate/sync-resources)).

**Trigger.** It can be push-button: a pending sync waits for confirmation. Turn off `auto_update` ("Automatically redeploy when newer image digests are available", off by default) ([Compose stacks](https://komo.do/docs/deploy/compose)).

**Drift.** The pages read describe diffing Komodo's *resources* against TOML and redeploying on git push. They don't describe continuously correcting running containers back to the Compose file.

**Secrets.** Variables and secrets live in the Komodo database, in Core's config file, or in a Periphery config file, and are interpolated with `[[KEY]]`. Komodo's docs say it "is not intended as an enterprise level" secret manager and suggest a dedicated tool alongside it ([Variables and Secrets](https://komo.do/docs/configuration/variables)). No 1Password integration is documented in the pages read.

**Fit.** It's a strong UI and a mature project, with the stock Compose file in git. However, Komodo holds secrets, and Core is a second control plane that would sit next to brokkr's Control Plane.

### Portainer git stacks

**What it is.** A management UI for Docker, Swarm, Podman and Kubernetes ([Add a new stack](https://docs.portainer.io/user/docker/stacks/add)). A Portainer Server talks to agents. The Edge Agent variant "polls the Portainer Server periodically to see if there are any pending jobs", so no ports need to be open on the host ([Edge Agent](https://docs.portainer.io/admin/environments/add/docker/edge)).

**Source and trigger.** A stack can come from "a docker-compose format file hosted in a Git repository". **GitOps updates** use either polling at a fetch interval or a webhook URL ([Add a new stack](https://docs.portainer.io/user/docker/stacks/add)).

**Drift.** "Force redeployment" redeploys the stack at each interval or webhook, "overwriting any changes that have been made in the local environment, even if there has been no update to the stack in Git" ([Add a new stack](https://docs.portainer.io/user/docker/stacks/add)). That's the bluntest and most complete drift correction among the Compose tools, at the cost of recreating containers on every interval.

**Trigger caveat.** "Re-pull image" is "equivalent to the `--pull=always` flag" ([Add a new stack](https://docs.portainer.io/user/docker/stacks/add)). Combined with mutable tags, it turns new images into Deploys.

**Secrets.** Stack environment variables are entered in Portainer ([Add a new stack](https://docs.portainer.io/user/docker/stacks/add)). No 1Password integration was found in the pages read.

**Fit.** It's the most adopted Compose GitOps option, with a stock Compose file in git. Portainer Server is a central component, and secrets are held in Portainer unless something else injects them.

## Docker Swarm

**What it is.** Orchestration built into Docker Engine. Managers keep "the desired state of the swarm", and "if a worker node becomes unavailable, Docker schedules that node's tasks on other nodes" ([Swarm key concepts](https://docs.docker.com/engine/swarm/key-concepts/)). Changing a service makes Docker "stop the service tasks with out of date configuration, and create new ones matching the desired configuration" ([Swarm key concepts](https://docs.docker.com/engine/swarm/key-concepts/)). One node can be both manager and worker ([Swarm key concepts](https://docs.docker.com/engine/swarm/key-concepts/)), and `docker swarm init` is enough to start ([Deploy a stack](https://docs.docker.com/engine/swarm/stack-deploy/)).

**Format.** A stack file in Compose format, applied with `docker stack deploy --compose-file compose.yaml <name>`. Swarm is the closest Docker analogue to a Deployment plus its controller. One catch: `docker stack deploy` "uses the legacy Compose file version 3 format", and the current Compose Specification "isn't compatible" ([Deploy a stack](https://docs.docker.com/engine/swarm/stack-deploy/)). A Swarm Platform would render a different dialect from a Compose Platform.

**Drift.** Task-level reconciliation is built in. Git-level reconciliation is not. Swarm doesn't read git, so it needs Doco-CD ("Swarm stacks in Swarm mode", [README](https://github.com/kimdre/doco-cd/blob/main/README.md)), Komodo or Portainer on top.

**Secrets.** A Swarm secret is created with `docker secret create`, then "stored in the Raft log, which is encrypted" and mounted in-memory at `/run/secrets/` ([Manage sensitive data](https://docs.docker.com/engine/swarm/secrets/)). A stack can mark secrets `external: true` ([Manage sensitive data](https://docs.docker.com/engine/swarm/secrets/)), but the `environment` source "is not supported when deploying with docker stack deploy" ([Compose secrets reference](https://docs.docker.com/reference/compose-file/secrets/)). The swarm therefore holds a copy of every secret, so something must create and rotate those secrets from 1Password. Doco-CD deploys Swarm stacks and has a 1Password provider ([README](https://github.com/kimdre/doco-cd/blob/main/README.md)). Whether the two combine for Swarm secrets wasn't checked.

**Fit.** Swarm has a real reconciler and a familiar format. It also has a legacy-format restriction, holds secrets, low visible adoption, and a future that rests largely on Mirantis's 2030 commitment ([Mirantis](https://www.mirantis.com/blog/mirantis-guarantees-long-term-support-for-swarm/)).

## Podman Quadlet (systemd)

**What it is.** Podman ships a systemd generator. Files such as `.container`, `.volume`, `.network`, `.pod`, `.kube` and `.image` in `/etc/containers/systemd/` (or per-user paths) "are read during boot (and when `systemctl daemon-reload` is run) and generate corresponding regular systemd service unit files" ([podman-systemd.unit(5)](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)). Options in `[Service]` and `[Install]` "pass directly to systemd" ([podman-systemd.unit(5)](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)), so restart behaviour is ordinary systemd `Restart=`.

**Format, two options.**

- **Quadlet units:** one `.container` file per container. Keys map onto `podman run` flags, for example `AddDevice=` maps to `--device` (GPU passthrough), `Secret=` to `--secret`, and `AutoUpdate=registry` to the auto-update label ([podman-container.unit(5)](https://docs.podman.io/en/latest/markdown/podman-container.unit.5.html); [podman-systemd.unit(5)](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)).
- **Kubernetes YAML:** a `.kube` unit "deploys containers from Kubernetes YAML" ([podman-systemd.unit(5)](https://docs.podman.io/en/latest/markdown/podman-systemd.unit.5.html)) through `podman kube play`, which supports the kinds Pod, Deployment, PersistentVolumeClaim, ConfigMap, Secret, DaemonSet and Job ([podman-kube-play(1)](https://docs.podman.io/en/latest/markdown/podman-kube-play.1.html)). Service and Ingress aren't on that list, so exposing traffic through a Load Balancer would need another mechanism.

**Apply.** Drop files in, run `systemctl daemon-reload`, then start or restart the units. These are stock tools on any systemd distribution with Podman.

**Drift.** systemd restarts failed units, and units come back at boot. Nothing compares the files with git, and an edited unit file isn't noticed until the next `daemon-reload`.

- **FetchIt** is the Podman GitOps agent. It defines "containers, pods, host services, or files in Git, and FetchIt reconciles them on a configured schedule", including Quadlet bundles with "update/removal handling" ([README](https://github.com/containers/fetchit/blob/main/README.md)). It's small (178 stars), says it's "under active development", and says new features on `main` are "unreleased until included in a published release". It requires Podman 5 and says "Podman 6 is not supported by Quadlet" ([README](https://github.com/containers/fetchit/blob/main/README.md)).
- **`podman auto-update`** isn't a git agent. With `AutoUpdate=registry`, it pulls when the registry digest differs from the local one and restarts the unit, on a daily systemd timer, and rolls back if the restart fails ([podman-auto-update(1)](https://docs.podman.io/en/latest/markdown/podman-auto-update.1.html)). That's automatic on image change, the opposite of push-button.

**Secrets.** Podman secrets use a driver: `file`, `pass` (GPG), or `shell`, where "secret is managed by custom scripts" over stdin/stdout ([podman-secret-create(1)](https://docs.podman.io/en/latest/markdown/podman-secret-create.1.html)). A `shell` driver whose lookup script calls `op read` could resolve 1Password references at container start. That's an inference from the driver interface, not a documented integration. FetchIt supports SOPS-encrypted YAML ([README](https://github.com/containers/fetchit/blob/main/README.md)).

**Fit.** systemd is a solid, stock reconciler for process liveness, and both formats are stock. The `.kube` path would let one Kubernetes-YAML renderer serve both Platforms for the supported kinds. The git-pull piece is the weak link: either brokkr's Agent writes the files and reloads systemd, or it relies on an early-stage FetchIt.

## Kamal (additions to render-vs-apply)

render-vs-apply covers Kamal's push-over-SSH model, `kamal-proxy` cutover and its 3-day rollback limit. The additions for #16:

- **Drift:** Kamal has no agent ([render-vs-apply, Kamal](https://github.com/worldtree-industries/brokkr/blob/research/render-vs-apply/docs/research/render-vs-apply.md#kamal)). Only Docker's restart policy runs between Deploys.
- **Secrets:** Kamal reads `.kamal/secrets` (or `.kamal/secrets.<destination>`) through dotenv ([Environment variables](https://kamal-deploy.org/docs/configuration/environment-variables/)). `kamal secrets fetch --adapter 1password --from Vault/Item …` pulls values through the 1Password CLI by command substitution in that file ([kamal secrets](https://kamal-deploy.org/docs/commands/secrets/)). References stay in git and are resolved where `kamal` runs, which is brokkr's Control Plane, not the Target.
- **Format:** `config/deploy.yml`, merged with `config/deploy.<destination>.yml` for `-d <destination>` ([Configuration overview](https://kamal-deploy.org/docs/configuration/overview/)). It describes how to deploy, not a full Desired State, and it can only be applied with Kamal itself, a Ruby gem rather than a stock Docker tool.

## Nomad (additions to render-vs-apply)

render-vs-apply covers the jobspec, `plan` → `run -check-index` and Raft snapshots. The additions for #16:

- **Drift:** the Docker driver handles "starting, watching, and cleaning up after containers" ([Docker driver](https://developer.hashicorp.com/nomad/docs/job-declare/task-driver/docker)), so liveness drift is handled the way Kubernetes handles it. Nomad doesn't read git, so git-level drift needs an outside loop ([render-vs-apply, Nomad](https://github.com/worldtree-industries/brokkr/blob/research/render-vs-apply/docs/research/render-vs-apply.md#nomad)).
- **Secrets:** Nomad variables are stored "directly in Nomad's state store", encrypted and replicated over Raft ([Variables](https://developer.hashicorp.com/nomad/docs/concepts/variables)). Nomad holds them, unless templates fetch from an outside store such as Vault.
- **Licence:** Nomad 1.7.0 and later is under the Business Source License, with production use allowed except to compete with IBM's paid versions ([LICENSE](https://github.com/hashicorp/nomad/blob/main/LICENSE)).
- **Weight:** it turns the host into a Nomad cluster ([render-vs-apply, Nomad](https://github.com/worldtree-industries/brokkr/blob/research/render-vs-apply/docs/research/render-vs-apply.md#nomad)), a second orchestrator to operate next to Kubernetes.

## Other approaches

- **Single-node k3s.** "A fully compliant Kubernetes distribution", "distributed as a single binary", aimed at edge, homelab and single-board machines ([K3s docs](https://docs.k3s.io/)). On the local-LLM box this would make the box a Kubernetes Target. brokkr's Kubernetes Platform and Agent would cover it, and no second Platform would be needed for that case.
- **Self-hosted PaaS (Coolify, Dokploy, Dokku, CapRover).** These have the most stars outside Docker itself (see "Adoption"). They're Heroku-style platforms that build and run apps on your own servers. Each has its own control plane and database, so they compete with brokkr's Control Plane rather than serving as its applier. Their docs weren't read in depth for this note.
- **Watchtower.** It used to be the standard "pull new images automatically" sidecar. The original repo is archived and "no longer maintained" ([README](https://github.com/containrrr/watchtower/blob/main/README.md)). An active fork exists ([nicholas-fedor/watchtower](https://github.com/nicholas-fedor/watchtower), 4,884 stars). It's automatic on image change, which conflicts with push-button.
- **Ansible or CI over SSH running `docker compose up`.** This is common in practice, and Ansible sits at 12.0% in SO 2026 ([SO 2026](https://survey.stackoverflow.co/2026/technology/data)). It's push-style with no reconciler, the same shape as Kamal.

## Comparison against #16's constraints

Legend: ✅ fits as documented, ⚠️ fits with extra work or caveats, ❌ doesn't fit.

| Option | Desired State format | Rebuild by hand with stock tools | Agent corrects drift | Push-button | Secrets referenced, not held | Mixed Environment (k8s + this) |
|---|---|---|---|---|---|---|
| **Compose, bare** | Compose file | ✅ `op run -- docker compose up -d` | ❌ restart policy only | ⚠️ needs something to run `up` | ✅ `op://` via `op run` | ✅ separate Target |
| **Compose + Doco-CD** | Compose + `.doco-cd.yml` | ✅ Compose part; ⚠️ secret mapping is Doco-CD config | ⚠️ event-driven (missing/unhealthy) plus git poll | ✅ commit = Deploy; sync windows | ✅ 1Password provider resolves `op://` | ✅ |
| **Compose + Komodo** | Compose + Komodo TOML | ✅ Compose part | ⚠️ resource diff; container drift not documented | ✅ manual sync confirm | ❌ held in Komodo DB/config | ⚠️ second control plane |
| **Compose + Portainer** | Compose | ✅ | ✅ "Force redeployment" each interval (recreates) | ⚠️ webhook/poll; keep "Re-pull" off | ❌ env held in Portainer | ⚠️ second control plane |
| **Docker Swarm** | Compose v3 (legacy) stack | ✅ `docker stack deploy` | ✅ tasks; ⚠️ git needs agent on top | ⚠️ needs agent or trigger | ❌ held in Raft (`external` must pre-exist) | ✅ |
| **Podman Quadlet** | `.container` units, or k8s YAML via `.kube` | ✅ copy files + `systemctl daemon-reload` | ✅ systemd liveness; ⚠️ git via FetchIt (early) or own Agent | ⚠️ keep `AutoUpdate=` off | ⚠️ `shell` secret driver + `op read` (inferred) | ✅; `.kube` can share k8s renderer |
| **Kamal** | `deploy.yml` (Kamal-only) | ❌ needs Kamal | ❌ no agent | ✅ native | ✅ resolved at deployer | ✅ |
| **Nomad** | HCL jobspec | ⚠️ needs Nomad cluster | ✅ allocations; ⚠️ git needs outside loop | ✅ `plan`/`run -check-index` | ❌ variables held in Raft (or Vault) | ⚠️ second orchestrator |
| **Single-node k3s** | Kubernetes YAML | ✅ `kubectl apply` | ✅ same as k8s | ✅ same as k8s | ✅ same as k8s | ✅ becomes a k8s Target |

## Implications for brokkr's non-Kubernetes Platform

These are options for the grilling ticket, not decisions.

### 1. Which native format the Desired State holds

- **(A) A Compose file per Deployable or per Environment on that Target.** This is the most widely known format (see "Adoption") and the native input of Doco-CD, Komodo and Portainer. Pin images by digest. For a by-hand rebuild, git also needs an env file of `op://` references. Keep to the Compose Specification, not Swarm's v3, unless Swarm is chosen.
- **(B) Quadlet `.container` units.** These are stock on systemd hosts, and systemd does liveness. There's one file per container, plus `.network` and `.volume` units.
- **(C) Kubernetes YAML, run by `podman kube play` (`.kube` units) or by k3s.** One renderer can serve both Platforms. With Podman, only the supported kinds apply (no Service or Ingress). With k3s, the "non-k8s" Target simply becomes a Kubernetes Target.
- **(D) A Swarm stack or Nomad jobspec.** You get a built-in reconciler, but at the cost of a legacy Compose dialect (Swarm) or a second orchestrator (Nomad), and both hold secrets.

### 2. What the Agent would run on the Target

- **(i) brokkr's own Agent.** It pulls the Desired State from git, runs `op run -- docker compose up -d --remove-orphans --wait` (or writes Quadlet files and reloads systemd), and repeats on an interval to correct drift. Its steps are exactly the by-hand rebuild, which keeps the "stock tools alone" promise honest.
- **(ii) An existing agent as the Agent.** Doco-CD is the closest fit: on-host, git-pull, `op://` references, Compose and Swarm, sync windows. The other options are Portainer Edge Agent (strong drift correction, central server), Komodo Periphery (driven by Core) and FetchIt (Podman, early). Choosing one means brokkr's Control Plane only writes git, the same shape as Flux on Kubernetes ([render-vs-apply §1](https://github.com/worldtree-industries/brokkr/blob/research/render-vs-apply/docs/research/render-vs-apply.md#1-where-the-button-is-decides-whether-git-tells-the-truth)).
- **(iii) No agent (Kamal or SSH push).** This drops the "Agent continuously corrects drift" constraint for this Platform.

### 3. Drift: "continuously" means something weaker on a host

No Compose tool does a Kubernetes-style field-by-field reconcile on a loop. The real choices are re-running `up` on an interval (idempotent: it only recreates changed services), Portainer-style forced redeploys, or event-driven repair (Doco-CD). systemd and Swarm repair crashed processes but not edits made by hand. The grilling should decide what "drift" means for this Platform: dead containers, hand-edited containers, or a host that no longer matches git.

### 4. Push-button

The model from render-vs-apply carries over: the Control Plane's git commit is the Deploy, and the Agent applies it. Every image-watching feature must be off, including Komodo `auto_update`, Portainer "Re-pull image", Podman `AutoUpdate=` and Watchtower. Images must be pinned by digest, or a new tag becomes a Deploy without anyone pressing the button.

### 5. Secrets

The pattern that keeps secrets referenced and stays rebuildable by hand is: `op://` references in git, resolved on the Target at apply time by the 1Password CLI (`op run` or `op inject`) or by an agent with a 1Password provider (Doco-CD). Each Target then holds one scoped service-account token ([Service Accounts](https://developer.1password.com/docs/service-accounts/)). Swarm, Nomad, Komodo and Portainer all store values themselves, so they'd need a sync step that copies values into them, which breaks the "never held" preference.

### 6. Mixed Environments

Every option can sit next to `togekiss` as a separate Target, because the Platform split already sits above the applier ([render-vs-apply §3](https://github.com/worldtree-industries/brokkr/blob/research/render-vs-apply/docs/research/render-vs-apply.md#3-keeping-the-platform-seam-open)). For the local-LLM box specifically, the choice is between a Docker or Podman Target with GPU passthrough in the file (Compose `devices` or Quadlet `AddDevice=`) and a single-node k3s Target that reuses the Kubernetes Platform. The existing Compose host `porygon` ([talos-k8s-bootstrap `docker-compose/`](https://github.com/dfryer1193/talos-k8s-bootstrap/tree/main/docker-compose)) is a real example of the Compose shape already in the estate.

## Open questions for the grilling

- Is the second Platform "Docker host" (Compose), "Podman host" (Quadlet), or neither, with k3s covering single boxes?
- Is brokkr's Agent its own small loop, or an adopted tool such as Doco-CD? If adopted, does its extra config (for example `.doco-cd.yml`) count as part of the Desired State?
- How does a Load Balancer reach a Deployable on a Docker host? Kamal uses `kamal-proxy`. Compose setups typically run a reverse proxy container, which wasn't researched here.
