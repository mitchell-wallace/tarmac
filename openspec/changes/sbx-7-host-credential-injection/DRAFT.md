```md
## Proposal: Host-to-Sandbox Credential Injection for Agent CLIs

## Summary

Today, authenticating `claude`/`codex`/`opencode`/`gh` inside a `dunex` sandbox means logging in interactively once per profile; the credential then survives sandbox recreation only because it lands in the profile's host-side persist directory, which the sandbox symlinks into. This is a report, not a proposal: it surveys what exists today, surfaces a mechanism already built into `sbx` itself that goes further than Dune currently uses, and lays out options for a future change that would let credentials be provisioned from the host and reused across sandboxes without ever writing raw long-lived secrets into sandbox-visible disk.

This document is a first pass for future agents/proposals to pick apart, not a committed design. It intentionally stops short of a `proposal.md`/`design.md`/`tasks.md` set.

## Depends On

```text
sbx-3-sbx-runtime-backend   (persist dir + secrets.go runner seam)
sbx-4-sbx-network-and-secrets   (secrets posture, D5)
```

## Problem

`container/base/scripts/setup-persist.sh` seeds `PERSIST_DIR` (`/persist/agent`, backed by `~/.local/share/dune/persist/<profile>` on the host) and symlinks `~/.claude`, `~/.codex`, `~/.config/opencode`, `~/.config/gh`, `~/.gitconfig`, `~/.git-credentials`, `~/.claude.json` into it. Once a user runs `claude login` / `gh auth login` inside a sandbox, the resulting session files land in that persisted, host-backed location, so future launches of the *same profile* skip re-login. This works, but:

- it requires one interactive login **per profile**, not once per host;
- the raw session credential (an OAuth bearer token, a GitHub PAT, etc.) sits as a plaintext file inside a directory that is fully mounted and readable by the sandboxed agent process — there is no separation between "the agent can make authenticated API calls" and "the agent (or anything it runs) can read the raw token";
- `internal/dune/runtime/sbx/secrets.go` already wraps `sbx secret set/ls/rm`, but its own doc comment says plainly: *"No core boot path sets a service secret in v1 — this is a forward-looking, tested surface, not wired into up/Ensure."* Nothing calls it today except tests.
- `sbx-4`'s design.md (D5) explicitly deferred this: *"Agent-provider credentials continue to use persisted config under the profile-scoped `/persist/agent` location ... unless/until a built-in agent or kit declares a service identifier suitable for `sbx secret` injection."* That condition already looks satisfied (see Findings below) but nobody has followed up.

## Objectives

This document should:

- describe precisely how `sbx`'s own secrets/proxy mechanism works, since it is more capable than Dune currently assumes;
- compare it against the current persist-dir approach on the specific axis the user asked about — how injected credentials would *persist*;
- lay out concrete options a future proposal could choose between;
- flag what is verified fact (from reading `sbx --help` output and repo source on this host) versus what needs to be checked against a live `sbx` daemon before any implementation.

## Non-Goals

This document should not:

- implement any credential injection;
- change `setup-persist.sh` or the current symlink behavior;
- pick a final mechanism;
- assume the `sbx` CLI surface below is stable across `sbx` versions (the `sbx-4` design doc already flags this: "reconfirm via `sbx secret --help` before use").

## Findings: what `sbx` already provides

Run on this host (`sbx secret --help`, `sbx secret set --help`, `sbx secret import --help`) — this is a live third-party CLI surface, not something in this repo, so treat it as an external dependency that can drift:

- **`sbx` runs its own egress proxy per sandbox.** For a fixed set of *service identifiers* — `anthropic, cursor, droid, github, google, groq, mistral, nebius, openai, openrouter, xai` — the proxy authenticates outbound API requests **on the sandbox's behalf**. Per the CLI's own description: *"the secret is never exposed directly"* to the sandboxed process/filesystem. This is a materially stronger isolation boundary than the persist-dir symlink approach, and it's the *intended* answer to "how do secrets get into a sandbox" per `sbx-2` D7 / `sbx-4` D5 ("no secrets in the template", "prefer service-identifier secrets").
- **Secrets can be scoped globally (`-g`) or per-sandbox.** Global service secrets are described as available to "every new sandbox" (stated explicitly for registry secrets in `sbx secret --help`; the service-secret help text uses the same "global vs sandbox" language but doesn't spell out the "every new sandbox" guarantee as explicitly — **unverified**, worth confirming before relying on it).
- **Three ways to populate the keychain:**
  1. `sbx secret set [-g|SANDBOX] <service> -t <token>` — direct API-key/token entry.
  2. `sbx secret import [SERVICE] [--all|--force|--dry-run]` — auto-detects host env vars (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GH_TOKEN`, etc.) and imports them into the **global keychain**.
  3. Running the agent's own interactive login *inside a sandbox*, e.g. `sbx run claude ... -- auth login` — sbx captures the resulting OAuth token into its keychain too. Per `sbx secret import --help`: *"Services that already have an OAuth token configured ... are skipped: the OAuth token takes precedence at runtime."* So sbx already has a notion of "the real interactive login wins over an imported API key," and once captured this way, that OAuth token is reusable for future sandboxes without logging in again.
- **`--oauth` as a flag on `sbx secret set` is currently documented as "openai/global only."** There's no evidence of an equivalent `sbx secret set -g anthropic --oauth` flow for capturing a Claude subscription OAuth session directly (as opposed to via path 3 above, which does capture it, just via a live login rather than a flag). This distinction matters: `ANTHROPIC_API_KEY`-based auth is **console API-key billing**, not the same account/billing path as a `claude login` subscription session. A future design needs to be explicit about which of these two credential types it's provisioning.
- **Storage location and encryption-at-rest — now VERIFIED live** (2026-07-12, bare-VM host with real `sbx` installed, no OS keychain daemon running): `sbx secret set -g anthropic` printed *"No keychain detected - this secret will be stored in an encrypted file on disk"* and wrote to `~/.config/com.docker.sandboxes/com.docker.sandboxes/sandboxes/<base64 of "docker/sandbox/credentials/_/anthropic">/secretpass` (a base64-encoded key-path scheme, one directory per service). `file` identifies it as `age encrypted file, scrypt recipient (N=2**18)` — the `age` format with a scrypt-derived passphrase key, a real work-factor KDF, not a rot13-grade placeholder. Confirms `sbx` prefers an OS keychain when present and falls back to strong file-based encryption otherwise; Dune would be delegating custody to a genuinely encrypted-at-rest store, not a plaintext file. (Test secret was removed after: `sbx secret rm -g anthropic`.)
- **Coverage looks good against what Dune actually ships.** `container/base/tooling.yaml` ships `claude`, `codex`, `opencode` (plus `rally`/`laps`/`agy`/`thenn`, which aren't provider-auth CLIs in this sense). `sbx`'s service list (`anthropic, openai, google, groq, mistral, nebius, openrouter, xai, github`) maps cleanly onto `claude`→anthropic, `codex`→openai, and `opencode`'s multi-provider routing (which supports most of that same list) — plus `gh`→github for the persist-dir's `.config/gh`/`.git-credentials` case. This is more coverage than the current "just use the persist dir for everything" approach might suggest was possible.

## Options

### Option A — Delegate to `sbx`'s global keychain (probable direction, needs verification first)

Wire a `dune`-level command (or a first-run prompt) that shells out to `sbx secret import` / `sbx secret set -g <service>` for the services Dune's shipped agents actually use, using the existing `internal/dune/runtime/sbx/secrets.go` runner seam (currently unused — see Problem).

**Persistence:** lives in `sbx`'s own global keychain, entirely outside any Dune-managed directory, keyed by service name rather than by Dune profile. Survives `dune down`, sandbox recreation, and even deleting a profile's persist dir, because it isn't stored there at all. Revocation is `sbx secret rm -g <service>`, a single host-side action that immediately affects every future sandbox.

**Pros:** raw secret never enters the sandbox filesystem (the actual goal); reuses a lifecycle `sbx-4`'s own spikes found "clean" for service secrets (unlike custom secrets); one login benefits every profile/workspace, not just one.

**Cons / open questions:** global scope means all Dune profiles on a host would share one Anthropic/OpenAI identity unless per-sandbox scoping is used instead (trading persistence for isolation — see below); the OAuth-vs-API-key distinction above needs a product decision; whether global secrets actually reach `sbx create`-launched sandboxes (Dune's path) the same way they reach `sbx run`-launched ones is unverified.

### Option B — Extend the persist-dir copy-in, mirroring `rally_seed.go`

`internal/dune/runtime/sbx/rally_seed.go` already copies the host's own `~/.config/rally` into a fresh profile's persist dir on first boot (`seedHostRallyConfig`). The same pattern could copy `~/.claude`, `~/.config/gh`, etc. from the host into the persist dir, so a profile is pre-authenticated from whatever the host CLI is already logged into.

**Persistence:** unchanged from today — still the per-profile host-side persist dir, just auto-populated instead of requiring a fresh in-sandbox login.

**Pros:** minimal new code; directly reuses a pattern already reviewed and shipped; keeps today's per-profile isolation (each profile can carry a different identity).

**Cons:** this is the copy-a-raw-secret-into-the-sandbox-filesystem approach the `sbx` proxy model exists specifically to avoid — the same risk class as the GitHub-token-in-image issue this repo already fixed once, just at runtime instead of build time (`6f910bf`, `202c817`). Also assumes the *host itself* has the CLI installed and logged in, which won't be true for every user.

### Option C — Hybrid (likely realistic end state)

Use Option A's `sbx secret` injection for anything with a service identifier (anthropic/openai/google/groq/mistral/openrouter/xai/github), and keep the current persist-dir + in-sandbox-login path as the fallback for anything without one. This avoids a flag day and matches how `sbx-4` D5 already phrased the deferral ("unless/until ... a service identifier is available" — implying persist-dir is the fallback, not the exclusive mechanism, once a service identifier exists).

## Open Questions

- Does a global (`-g`) service secret actually get injected into sandboxes Dune creates via `sbx create`, or only ones created via `sbx run`? **Partially de-risked (2026-07-12), not fully closed**: `sbx create --help` and `sbx run --help` share an identical flag surface, and `run`'s own description is *"creating the sandbox if it does not already exist"* — i.e. `run` = `create` + attach, not a separate creation path. `sbx secret --help` frames injection generically as happening *"when a sandbox starts,"* not as a `run`-specific behavior. This is strong documentary evidence the two share one creation/injection path, but no template image was pulled on this host (fresh bare VM, no cached `sbx` templates, and pulling one felt like overreach for a side-verification) so it was not confirmed by actually booting a sandbox and inspecting proxy state inside it. Still worth a live trace before Option A is implemented.
- ~~Where and how does `sbx` store its own keychain, and what's its encryption-at-rest story?~~ **RESOLVED — see Findings above**: `age`-encrypted file store (scrypt KDF, N=2**18) under `~/.config/com.docker.sandboxes/...`, used as the fallback when no OS keychain daemon is present.
- Is the proxy's request-rewriting transparent regardless of what the sandboxed CLI process locally believes about its own auth state, or does the CLI still need some local marker (e.g. a placeholder env var) to route requests through a path the proxy can intercept? **Still open** — closing this needs a live sandbox with a service secret set, then inspecting its env/network config from inside (e.g. `sbx exec <name> -- env` and checking for injected proxy vars/CA trust) to see whether the agent CLI needs any awareness of the proxy at all. Not attempted this pass (same template-pull cost tradeoff as above).
- Product decision: should Dune provision Anthropic API-key auth (console billing) or Claude subscription OAuth (via a one-time `... auth login` inside a sandbox, captured by sbx) — or offer both and let the user choose per profile?
- UX shape: a `dune auth <service>` wrapper, an automatic first-run prompt on `dune up`, or just documentation pointing at `sbx secret import`?
- Does global scope's cross-profile sharing conflict with any existing expectation that profiles are isolated identities? (Today's persist-dir model implies per-profile isolation by construction; Option A would be a behavior change, not just an optimization.)

## Acceptance Criteria (for this draft stage)

- Findings above are cited against real `sbx --help` output and this repo's source, not assumed.
- At least three concrete options are laid out with their persistence model stated explicitly, since that was the specific question this document exists to answer.
- No code changes are made.
- Every claim about live `sbx` behavior not directly observed on this host is marked unverified.

## Risk Areas

### Treating `sbx --help` text as a stable contract

The exact flags/scoping semantics may drift across `sbx` releases (the `sbx-4` design doc already makes this caveat for policy/secret commands). A real proposal must reconfirm against the `sbx` version actually pinned/shipped before writing fakeRunner tests against these shapes.

### Cross-profile secret leakage (Option A)

If global scope is adopted without thought, two unrelated Dune profiles/workspaces on the same host would silently share one Anthropic identity. This may be desirable (one human, one subscription) or may not (a user intentionally separating work/personal profiles).

### Reintroducing a raw-secret-in-image/sandbox mistake (Option B)

This repo has already had to walk back baking a GitHub token into build-time images (`6f910bf`, on top of `main`'s `595c2d2`). Copying live session tokens into every sandbox's runtime filesystem is the same category of exposure at runtime instead of build time, and should be weighed against that history rather than treated as a fresh, unrelated decision.
```
