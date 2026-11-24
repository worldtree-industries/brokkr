---
status: accepted
---

# Brokkr builds its own GitOps loop, with Platform-native Desired State

Brokkr is split into a Control Plane and one Agent per Target, with no network API between them. The Control Plane renders a Release, through a per-Platform Renderer, into Desired State written in the Platform's native format (plain Kubernetes YAML now, something like Compose later), and commits it to git: pressing Deploy *is* that commit. Each Agent pulls its Target's Desired State and continuously corrects drift through a per-Platform Applier. The Desired State must be applyable with stock tools alone (`kubectl apply`, `docker compose up`), so an Environment can be rebuilt after a disaster without a working Brokkr, and Brokkr takes over once it's running again. Decided in [Map: brokkr v1 → What brokkr outputs, and what applies it](https://github.com/worldtree-industries/brokkr/issues/6).

## Considered Options

- **Adopt Flux or Argo CD as the applier.** Rejected: we want to build it ourselves, behind a platform-agnostic API with selectable implementations. Neither tool reaches a plain Docker host anyway, and one Environment must be able to mix Platforms (`prod/togekiss` plus `prod/llm`).
- **Platform-neutral Desired State, rendered by the Agent.** Rejected: it keeps git cleaner, but it makes DR much harder, because nothing except Brokkr can apply it.
- **Brokkr applies directly and only records the Deploy in git.** Rejected: DR would then need Brokkr running first, and nothing would correct drift between Deploys.

## Consequences

- The Desired State may use only stock or already-installed resource types (for example `HTTPRoute`, `OnePasswordItem`), never Brokkr-specific CRDs, and it must contain no unresolved templating.
- Every new Platform costs two implementations: a Renderer and an Applier.
- On Kubernetes the Agent is the only writer to the Deployables it manages. People get read-only `kubectl` unless they hold an elevated devops role.
