# How brokkr can learn about Projects and Releases from GitHub

Research for [brokkr#3](https://github.com/worldtree-industries/brokkr/issues/3) (map: [brokkr#1](https://github.com/worldtree-industries/brokkr/issues/1)). Terms are from `GLOSSARY.md`. This note lays out options. It doesn't pick one.

Researched 2026-11-24 from GitHub Docs (API version `2026-03-10`) and from live, read-only calls against the `worldtree-industries` org.

## The question

Brokkr needs three things from GitHub:

1. **Which Projects exist** in `worldtree-industries`.
2. **Which Releases each Project has**: its `vX.Y.Z` tags.
3. **A Release's Manifest**: `.forge.yaml` at that tag.

Deploys are push-button, not automatic on tag (map #1, standing preferences), so brokkr only needs this when a person opens the Deploy screen or presses the button. It doesn't have to react the moment a tag appears.

## How Releases are tagged today

- The shared `go.yml` workflow's `release` job runs last. It runs only after `lint`, `test` and `affected` succeed and `images` succeeds or is skipped. It creates a **lightweight** tag through the REST API, with no `git push`: `gh api --method POST "repos/$REPOSITORY/git/refs" -f "ref=refs/tags/$TAG" -f "sha=$SHA"`, with `GH_TOKEN: ${{ github.token }}` (`worldtree-industries/workflows` `.github/workflows/go.yml`, `release` job).
- Ivaldi ADR-0002 (`docs/adr/0002-every-merge-releases.md`, remote `main`) says every merge to `main` releases, and the tag comes last "so a failed build leaves no tag behind". It also says "Tags pushed with `GITHUB_TOKEN` start no workflow runs".
- Each run creates **one** tag. That matters because of the three-tag webhook limit below.
- Observed: mimir's `v0.2.0` came from run `go / release` (success, 2026-11-22T01:56:20Z). `images` was skipped in that run, so a tag doesn't mean every Deployable has a new image. Ivaldi's `docs/brokkr-notes.md` already says to "track the latest semver tag per image". That lookup belongs to Harbor and is outside this note.

## Do webhooks and App events fire for a tag created with `GITHUB_TOKEN` through the API?

**What the docs say:**

- The `GITHUB_TOKEN` rule only covers **workflow runs**: "events triggered by the `GITHUB_TOKEN` will not create a new workflow run", except for `workflow_dispatch`, `repository_dispatch` and some `pull_request` activity. The stated reason is "to prevent you from accidentally creating recursive workflow runs" ([GITHUB_TOKEN](https://docs.github.com/en/actions/concepts/security/github_token); [Triggering a workflow from a workflow](https://docs.github.com/en/actions/how-tos/write-workflows/choose-when-workflows-run/trigger-a-workflow#triggering-a-workflow-from-a-workflow)). The wording says the *event is triggered* but creates no run. It says nothing about webhooks.
- The webhook reference lists these exceptions for `create`: it "will not occur when more than three tags are created at once", and there is a 25 MB payload cap. For `push`, "events will not be created for tags when more than three tags are pushed at once". Neither event has an exception for `GITHUB_TOKEN`, GitHub Actions or the API ([Webhook events and payloads: `create`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#create), [`push`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#push)). None of the webhook pages fetched for this note (about, types, best practices, App webhooks) mention `GITHUB_TOKEN`.
- The `create` event "occurs when a Git branch or tag is created", and the docs don't care how it was created. The `push` event's description ("when a commit tag is pushed") is written in terms of `git push`. Its payload has `created: true` when "this push created the ref" ([`push`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#push)).

**Conclusion and limits:** the docs don't suppress webhooks for `GITHUB_TOKEN`. The `create` event should fire for the `release` job's tag, at one tag per run. But **no doc says outright** that it is delivered, or that a `push` event fires for a ref created with `POST /git/refs` rather than `git push`. I couldn't check it here:

- I can't list org webhooks or their deliveries: `GET /orgs/worldtree-industries/hooks` returns 404 because the token lacks `admin:org_hook`. The org has no App installations (`GET /orgs/worldtree-industries/installations` returns `total_count: 0`).
- The public Events API shows neither mimir's `v0.2.0` tag nor the 2026-11-22 merge push that caused it, and it has no tag `CreateEvent` for any org repo. That proves nothing: the Events API "is not built to serve real-time use cases… event latency can be anywhere from 30s to 6h" ([Events](https://docs.github.com/en/rest/activity/events)), and it's missing the human push too.

**How to settle it cheaply, if a webhook option is picked:** add a webhook that subscribes to `create` and `push`, either on one Project or as a test App. After the next merge, check its *Recent deliveries*. Deliveries from the past 3 days are listed ([Redelivering webhooks](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/redelivering-webhooks)). I didn't do this, because creating hooks changes org configuration.

## Option A: GitHub App

- **Registration and install.** An org can register an App "Only on this account", then install it on **All repositories** or **Only select repositories**. If it has any repository permissions, it "will always have at least read-only access to all public repositories on GitHub" ([Registering a GitHub App](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/registering-a-github-app); [Installing your own GitHub App](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app)). The webhook docs list no plan restriction for App or org webhooks ([Types of webhooks](https://docs.github.com/en/webhooks/types-of-webhooks)), so they should work on GitHub Free.
- **Permissions.** The App needs read access to **Contents**, which covers the `create` and `push` events and reading files. It also needs **Metadata**, which covers the `repository` event and `GET /orgs/{org}/repos` ([`create`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#create), [`repository`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#repository); [Permissions required for GitHub Apps: Metadata, Contents](https://docs.github.com/en/rest/authentication/permissions-required-for-github-apps)).
- **Webhooks.** "Each GitHub App has a single webhook", which starts with no event subscriptions ([Types of webhooks](https://docs.github.com/en/webhooks/types-of-webhooks)). Every App gets `installation_repositories` (repositories added to or removed from the installation) without subscribing ([`installation_repositories`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#installation_repositories)). Receiving any of them needs a webhook URL that GitHub can reach ([Using webhooks with GitHub Apps](https://docs.github.com/en/apps/creating-github-apps/registering-a-github-app/using-webhooks-with-github-apps)). For brokkr on `togekiss`, that means a public HTTPS ingress. The App can turn webhooks off and be used only for authentication (same page).
- **API budget.** An installation token gets 5,000 REST requests an hour. Off Enterprise Cloud, that grows with repositories and users above 20, up to 12,500 ([REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api#primary-rate-limit-for-github-app-installations)). GraphQL gets 5,000 points an hour per installation, with the same scaling ([GraphQL rate limits](https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api#primary-rate-limit)).
- **Trade-offs.** Brokkr would hold a short-lived token rather than a long-lived PAT, so the credential isn't tied to a person. In exchange, brokkr has to store the App's private key. "Only select repositories" could itself be the list of Projects (see "Telling an Ivaldi Project from other repos").

## Option B: org webhook

- Org webhooks "can subscribe to events that happen in all repositories owned by the organization". Only an **organization owner** can create one. Each event type allows at most 20 webhooks ([Types of webhooks](https://docs.github.com/en/webhooks/types-of-webhooks)). `create`, `push` and `repository` are all available at `organization` level ([Webhook events and payloads](https://docs.github.com/en/webhooks/webhook-events-and-payloads)).
- It gives no API credential. Brokkr would still need a token, or would make unauthenticated calls, to list tags and fetch Manifests. It also needs a public HTTPS endpoint, the same as Option A.

## Things that apply to both webhook options

- **Delivery isn't guaranteed.** A delivery fails if brokkr is down or takes more than 10 seconds to respond. "GitHub does not automatically redeliver failed deliveries", so a missed event has to be recovered by polling deliveries and redelivering within 3 days ([Handling failed webhook deliveries](https://docs.github.com/en/webhooks/using-webhooks/handling-failed-webhook-deliveries); [Redelivering webhooks](https://docs.github.com/en/webhooks/testing-and-troubleshooting-webhooks/redelivering-webhooks)). A webhook-fed list of Releases therefore needs an API pass to fill gaps, for example after a disaster-recovery rebuild of brokkr. Since Deploys are push-button, webhooks would mostly be a cache-refresh signal.
- **Three-tag limit.** No `create` or `push` event fires when more than three tags arrive at once ([`create`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#create), [`push`](https://docs.github.com/en/webhooks/webhook-events-and-payloads#push)). `go.yml` creates one tag per run, but a bulk retag done by hand would be missed silently.

## Option C: query the APIs on demand, with no webhooks

### Authentication and budget

- Every Project is a public repo, so reads can be **unauthenticated**, at 60 requests an hour per IP address. A **PAT** (or an App's user token) gets 5,000 an hour. An **App installation** token gets 5,000 or more ([REST rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api)). With 6 repositories today, and one tags call per Project per Deploy screen, 60 an hour is tight but possible. Any token makes it comfortable.
- Secondary limits: at most 100 concurrent requests and 900 points a minute per REST endpoint ([REST rate limits: secondary](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api#about-secondary-rate-limits)).
- **Conditional requests.** Most endpoints return an `etag`. A `304 Not Modified` "does not count against your primary rate limit if… made while correctly authorized" ([Best practices: conditional requests](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api#use-conditional-requests)). This only works when authenticated. Observed: `GET /repos/worldtree-industries/mimir/tags` returns an `ETag`. GitHub's guidance is to prefer webhooks and, if you must poll, to poll on a schedule with conditional requests (same page, "Avoid polling").

### REST endpoints

| Need | Endpoint | Notes |
|---|---|---|
| Repos in the org | `GET /orgs/{org}/repos` | Response includes `topics` and `archived` ([Repos](https://docs.github.com/en/rest/repos/repos#list-organization-repositories)) |
| Releases (tags) | `GET /repos/{owner}/{repo}/tags` | Paginated ([Repos: list tags](https://docs.github.com/en/rest/repos/repos#list-repository-tags)) |
| Releases, filtered by prefix | `GET /repos/{owner}/{repo}/git/matching-refs/tags/v` | Returns every ref that starts with the prefix ([Git refs](https://docs.github.com/en/rest/git/refs#list-matching-references)) |
| Manifest at a Release | `GET /repos/{owner}/{repo}/contents/.forge.yaml?ref=vX.Y.Z` with `Accept: application/vnd.github.raw+json` | `ref` is "the name of the commit/branch/tag". The full feature set works up to 1 MB, and raw works up to 100 MB. Can return `304` ([Contents](https://docs.github.com/en/rest/repos/contents#get-repository-content)) |

Verified live: `contents/.forge.yaml?ref=v0.1.0` on mimir returns the file. Neither endpoint returns tags in semver order (GraphQL can order refs by `TAG_COMMIT_DATE` or `ALPHABETICAL`), so brokkr has to sort `vX.Y.Z` itself to find "latest".

### GraphQL

A single query can return every org repo with its tags and whether it has a Manifest. Through `object(expression:)` it can also return the Manifest's text at a given tag. Verified live, and the whole-org query **cost 1 point**:

```graphql
query {
  organization(login: "worldtree-industries") {
    repositories(first: 50) {
      nodes {
        name
        manifest: object(expression: "HEAD:.forge.yaml") { ... on Blob { byteSize } }
        refs(refPrefix: "refs/tags/", first: 100) { nodes { name target { oid } } }
      }
    }
  }
  rateLimit { cost remaining }
}
```

Fetching one Release's Manifest works the same way: `repository(owner:, name:) { object(expression: "v0.2.0:.forge.yaml") { ... on Blob { text } } }`. That returned mimir's Manifest. GraphQL requires authentication: 5,000 points an hour for a user, or 5,000 or more for an installation ([GraphQL rate limits](https://docs.github.com/en/graphql/overview/rate-limits-and-query-limits-for-the-graphql-api)).

### Events API (not suitable)

`GET /repos/{owner}/{repo}/events` supports ETag and `X-Poll-Interval`. But it holds at most 300 events from the last 30 days, with latency of 30 seconds to 6 hours ([Events](https://docs.github.com/en/rest/activity/events)). As observed above, it showed none of the tag creations.

## Telling an Ivaldi Project from other repos

Org repos today, from `GET /orgs/worldtree-industries/repos`: `gomad`, `mjolnir`, `mimir`, `ivaldi`, `workflows` and `brokkr`. All are public, and **none has a topic**.

| Signal | Observed today | Notes |
|---|---|---|
| **`.forge.yaml` on the default branch** | Present in `mimir`, `ivaldi` and `brokkr`. Absent from `gomad`, `mjolnir` and `workflows`, which all have `v*` tags. | Ivaldi owns this name (`internal/manifest/manifest.go`: `const FileName = ".forge.yaml"`). A tags-only check would wrongly include `mjolnir`, `workflows` and `gomad`. |
| **`.forge.yaml` at the Release tag** | Present in mimir `v0.1.0` and `v0.2.0` | The strictest test: "is this tag a Release of a Project?" It costs one Contents call, or a GraphQL `object(expression:)` per tag. |
| **Manifest has a Deployable** (`kind: http` or `job`) | `ivaldi` has only `kind: tool`. `brokkr` has `kind: http` but no tags yet. | Having a Manifest doesn't mean there's something to Deploy, so brokkr may also filter by Kind. |
| **Repository topic** (for example `ivaldi-project`) | None set | Repository admins add topics by hand. They're lowercase, at most 50 characters, at most 20 per repo ([Topics](https://docs.github.com/en/repositories/managing-your-repositorys-settings-and-features/customizing-your-repository/classifying-your-repository-with-topics)). They come back in `GET /orgs/{org}/repos`. Ivaldi doesn't set topics (the only code-search hit for "topic" in `ivaldi` is `docs/agents/domain.md`), so a topic could drift from the truth unless Ivaldi or the `go.yml` workflow started setting one. |
| **Custom property** | Org has no properties defined (`GET /orgs/{org}/properties/schema` returns `[]`, so the endpoint works on this Free org) | Structured, org-defined metadata. Visible to anyone on public repos ([Custom properties](https://docs.github.com/en/organizations/managing-organization-settings/managing-custom-properties-for-repositories-in-your-organization)). Set by hand like topics. |
| **Code search** `filename:.forge.yaml org:worldtree-industries` | Not tried | Searches the default branch only, files under 384 KB, authenticated, 10 requests a minute, with index lag ([Search code](https://docs.github.com/en/rest/search/search#search-code)). Adds nothing over checking each repo. |
| **App installed on "Only select repositories"** | No App | The installation's repository list becomes the set of Projects. `installation_repositories` reports changes ([Installing your own GitHub App](https://docs.github.com/en/apps/using-github-apps/installing-your-own-github-app)). It's a manual step, like topics. |

## Summary of options

| | A: GitHub App | B: org webhook | C: on-demand API |
|---|---|---|---|
| Learns of a new Release | Webhook (`create`), seconds after the tag. Delivery for `GITHUB_TOKEN` tags not yet verified. | Same as A | On the next query, for example when the Deploy screen opens |
| Missed events | Not redelivered automatically; needs an API pass to fill gaps | Same | No events to miss |
| Credential | App private key, short-lived installation tokens | None from the webhook; still needs a PAT or App, or goes unauthenticated | None (60/h), a PAT, or an App with webhooks off |
| Inbound network | Public HTTPS endpoint | Public HTTPS endpoint | None |
| Who sets it up | Org owner registers and installs the App | Org owner | Whoever issues the token |

A webhook option (A or B) has to do the same on-demand queries as C anyway, to recover from missed events and for disaster recovery. So C is the part every option shares, and webhooks are an optional speed-up on top.
