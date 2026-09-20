# GitLab connection

This checkout adds a GitLab source-control provider to Intent, including
self-managed instances such as `https://git.euraika.net`. The desktop renderer
and `intentd` must both be built from this checkout.

The connection model follows the [upstream architecture review](https://github.com/intent-hq/intent/issues/5466#issuecomment-5747251886): credentials are registered per instance and each repository resolves its own provider. GitHub and multiple GitLab instances can be used concurrently.

## Connect

In the new-workspace connection step, choose **GitLab**, or open
**Settings → Connections** and choose **GitLab** there. Enter the instance URL.
Use the instance's root URL, without `/api/v4`. GitLab installations under a URL
path are supported; keep that path in the instance URL.

Choose a credential source:

- **GitLab CLI** uses `glab config get token --host <configured-host>`. An existing
  login can be reused; the connector does not run `glab auth login` or modify the
  CLI's account configuration. A host-specific `glab auth status` check runs
  first so an unknown host cannot receive a fallback credential; glab can
  refresh an existing OAuth session during that check. OS keyring credentials
  are supported. Instances under a URL path require a stored token or
  a token bound to the complete instance URL through the environment.
- **Explicit token** uses a personal, group, or project access token saved in
  Intent's secret store. The token is bound to the configured instance URL.
- **Environment** uses `GITLAB_TOKEN` or `GL_TOKEN`. For self-managed servers,
  set `GITLAB_HOST` (or `GL_HOST`) to the configured instance, for example
  `git.euraika.net`. An unbound environment token is used only for GitLab.com.
- **Auto** tries the instance-bound stored token, environment, then `glab`.

GitLab API reads require `read_api`; writes such as opening or merging a merge
request require `api`. A token with only `write_repository` does not grant REST
API access. The user's or token's project role still controls what is allowed.

Save and test the connection. The connected account and instance identify the
server Intent will use. Then continue to the project step and choose **GitLab**
to search your repositories, including projects in nested groups. Selecting a
repository preserves its canonical URL on your GitLab instance. An existing
local checkout remains available through **Local**.

Private HTTPS clone/fetch/push uses the credential registered for that repository's instance. A scoped credential carries the HTTPS authority, port and installation prefix through Git operations. Credential-bearing operations use system Git with redirects disabled, including URL-specific redirect settings; local and SSH operations retain their existing transport. Shell helpers preserve existing helpers as fallbacks; SSH continues to use the user's SSH configuration. Terminals and agents receive a daemon helper, with no raw token in their environment. The helper resolves the credential over the local Unix socket for each request, so disconnecting takes effect in already-open terminals.

## Connections and repository identity

A connection ID is its canonical instance URL, for example `https://git.euraika.net` or `https://example.net:8443/gitlab`. The public registry contains its provider, token source, enabled state and child-credential setting. Each GitLab secret is a separate atomic record containing provider, full instance URL and token. A secret whose embedded identity does not match is rejected. GitHub keeps its existing dedicated secret slot for device-flow compatibility.

Repository operations select a connection from the checkout's origin. A stored PR/MR URL supplies provenance only when no origin is available. Unknown hosts require a configured connection; they never fall through to GitHub credentials. Installation prefixes use the longest registered match. PR monitors and rate-limit state are separated by connection. GitLab clone-cache slots include the full repository URL, preventing a second host with the same project slug from replacing a pending checkout source.

The historical `github.*` RPCs remain GitHub-specific. New `sourceControl.*` browse methods take an explicit `connectionId`, `repoUrl` or `workspaceId`; `pr.*` derives its connection from the workspace. The selected connection in a repository picker is a client preference, not a daemon-wide provider switch. `sourceControl.activeProvider` is retained only for legacy configuration compatibility and does not select a repository's provider.

Project, issue and MR links retain the original host and nested namespace. GitLab merge requests use project-local IIDs. The legacy workspace field is still named `githubUrl`, but accepts forge URLs through the same repository resolver.

## Merge requirements

GitLab snapshots include project pipeline/discussion policies, effective required approvals, the MR's head pipeline, job statuses and `allow_failure`. Optional jobs do not become required checks. Unavailable requirement data stays unknown; it is not reported as success. The composed REST observation reuses fetched payloads and bounded pagination; it does not claim GitHub GraphQL's single-roundtrip behavior.

Unsupported operations still return explicit errors: rebase merge selection, branch-update emulation, request-changes reviews, multi-project searches, and submitting an approval with a comment atomically. Merging sends the freshly read head SHA; the shared merge contract does not carry a SHA from the user's earlier review. Merge trains are not implemented by this connector.

## Wire configuration

Use the `sourceControl.connections.configure` RPC to save a connection and `sourceControl.authStatus` to check it. PATs stay in the connection form until submitted and never enter Redux or persisted renderer state. They are never returned by list/status calls. The `sourceControl.connections.disconnect` RPC disables only that connection. Legacy GitHub device-flow credentials and GitLab instance-bound token records remain readable during migration.

HTTPS is required outside loopback API fixtures. Credential-bearing API requests do not follow redirects. Git helpers retain the full path for installations under a prefix.

## Development and verification

Install the pinned Rust toolchain and frontend dependencies as described in the
repository development guide. From the monorepo root:

```sh
cd packages/intentd
cargo test -p intent-sourcecontrol
cargo test -p intentd --test e2e_wss_gitlab
cargo test -p intentd --test uds_git_credentials
cargo test -p intentd --test gitlab_https_transport
cd ../cloudlands-fe
corepack pnpm run check
```

The GitLab provider tests use a local HTTP fixture and exercise request paths,
credentials, pagination, merge requests, and failure handling. The WSS suite
drives the real TLS transport, router, settings registry, secret store, and
GitLab provider together. It does not mutate a live GitLab project.

Use the existing isolated `make dev-sandbox-stack` workflow for a browser
preview. Use the documented Electron development workflow to run the native
app. Do not point a test build at the installed app's database: newer upstream
code may include unrelated migrations.

## Upstream references

- [GitLab REST authentication](https://docs.gitlab.com/api/rest/authentication/)
- [Merge requests](https://docs.gitlab.com/api/merge_requests/)
- [Discussions](https://docs.gitlab.com/api/discussions/)
- [Pipelines](https://docs.gitlab.com/api/pipelines/)
- [Token scopes](https://docs.gitlab.com/security/tokens/access_token_scopes/)

The upstream public repository currently defers external pull requests; this
implementation can be maintained locally without changing the installed app.
