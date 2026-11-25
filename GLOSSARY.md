# Brokkr

Brokkr is worldtree's deploy system: it turns a Release built by GitHub into everything needed to run it in an Environment, so that developers never need to know what Platform their code runs on.

## Language

### What gets deployed

**Project**:
A repository scaffolded by Ivaldi, holding one or more Binaries and a Manifest (as defined in Ivaldi's glossary).

**Deployable**:
A Binary of a Kind that runs somewhere (`http` or `job`), as opposed to a Tool.
_Avoid_: Service, app, workload

**Manifest**:
The `.forge.yaml` file in a Project: its Binaries, their Kinds, and their Manifest Settings.
_Avoid_: Config, spec

**Release**:
A `vX.Y.Z` git tag of a Project at which one of its Deployables changed, together with the image built for that Deployable at that tag and published to the registry. Each Deployable has its own Releases: a tag that didn't change a Deployable is not one of its Releases.
_Avoid_: Version, build, image tag

### Where it runs

**Platform**:
A kind of runtime a Deployable can run on, such as Kubernetes or a plain Docker host.
_Avoid_: Backend, runtime

**Target**:
One running instance of a Platform within an Environment, named `<environment>/<name>`: for example `prod/togekiss` (a Kubernetes cluster) or `prod/llm` (a Docker host).
_Avoid_: Cluster, host, node

**Environment**:
One of the fixed stages defined in mjolnir (`dev`, `staging`, `prod`) that a Release is deployed into. An Environment is made up of one or more Targets.
_Avoid_: Cluster, stage

**Load Balancer**:
A named, org-defined entry point that a Deployable can be exposed through. A Manifest may name one, but never says what kind of load balancer it is.
_Avoid_: Gateway, ingress

### How it's shaped

**Org Default**:
The value the org applies to a Deployable when its Manifest says nothing.

**Manifest Setting**:
A value in a Project's Manifest that replaces an Org Default for one Deployable, stated without reference to any Platform.
_Avoid_: Override (Mimir uses that word for config values)

**Deploy**:
A single push-button action that puts one Release into one Environment.
_Avoid_: Rollout, sync, push

**Desired State**:
The fully resolved record in git of what runs on each Target, written in its Platform's native format so that the Platform's own tools can apply it without Brokkr.
_Avoid_: Deploy repo, manifests (Manifest means `.forge.yaml`)

### Brokkr's parts

**Control Plane**:
The part of Brokkr that owns the Deploy button: it resolves a Release with its Org Defaults and Manifest Settings and commits the result to the Desired State.
_Avoid_: Server, brokkr core

**Agent**:
The part of Brokkr that runs on one Target and continuously makes it match that Target's Desired State.
_Avoid_: Reconciler, operator, controller
