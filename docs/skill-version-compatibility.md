# Skill ↔ API version-compatibility spec (opt-in model)

How a Treasures skill (`treasures-wallet`, `treasures-b2b-api`) and the Treasures
API negotiate version compatibility so that a stale skill is warned and ultimately
stopped — **without breaking the generic, non-skill clients that also use this API.**

This is the spec for the version gate. The agent-facing summary lives in each skill's
`SKILL.md` (b2b: "Version & compatibility"; wallet: Behaviors #7); the backend obligations
live in the middleware spec (in `treasures-finance-backend`).

## Principle

The B2B API is a **generic public API** with many independent integrations, not just
skill-built agents. So compatibility is enforced on two separate tracks:

1. **Skill freshness — opt-in.** A caller that identifies itself as a skill client
   (by sending `X-Treasures-Skill-Version`) opts into a skill-specific gate: it gets
   deprecation warnings and, once past sunset, a hard `426`. **Sending the header is
   the opt-in.**
2. **Contract safety — universal.** *Every* caller, skill or not, is protected by
   ordinary public-API versioning: the path version (`/api/v1`, `/public/v1` →
   `/v2`) plus `Sunset`/`410` on retired versions. This never singles anyone out.

A caller that sends no skill header is simply a generic client: it is **never**
rejected for "not using the skill" — it relies on track 2 alone.

## Two protection tiers

| Caller | Sends `X-Treasures-Skill-Version`? | Protected by |
| --- | --- | --- |
| **Skill client** (`treasures-wallet` / `treasures-b2b-api`) | yes — the skill's fetch helper does it | Opt-in skill gate (`Deprecation`/`Sunset` → `426 + upgrade`) **and** standard versioning |
| **Generic integration** (bespoke client) | no | Standard versioning only (`/v1`→`/v2`, `Sunset`/`410`) |

## Request headers (skill → API) — OPTIONAL

| Header | Value | Notes |
| --- | --- | --- |
| `X-Treasures-Skill` | `treasures-wallet` \| `treasures-b2b-api` | which client |
| `X-Treasures-Skill-Version` | the skill's `metadata.version` (e.g. `1.0.0`) | semver; **presence opts the caller into the skill gate** |

These are **not required**. The skill's fetch helper attaches them so skill clients
opt in automatically; any other client may omit them with no penalty.

## Response headers (API → caller)

Informational headers are safe to send to everyone; the deprecation signals are sent
only to callers that opted in (sent a skill-version header) and are below the
recommended version.

| Header | Sent to | Meaning |
| --- | --- | --- |
| `X-Treasures-Api-Revision` | all | current contract revision, date form (e.g. `2026-09-01`) — informational |
| `X-Min-Skill-Version` | all | minimum skill semver the gate still accepts — informational for generic clients |
| `Deprecation` | opt-in, stale | RFC 9745 — the caller's skill version is inside the sunset window |
| `Sunset` | opt-in, stale | RFC 8594 — HTTP-date when this skill version stops working |
| `Link: <url>; rel="sunset"` | opt-in, stale | upgrade documentation |
| `Warning: 299 - "<text>"` | opt-in, stale | human-readable deprecation notice |

## The opt-in gate

Applies **only** when the request carried `X-Treasures-Skill-Version`:

- **Version ≥ `X-Min-Skill-Version`** → normal response (plus the informational headers).
- **Version below floor, inside the window** → success **+** `Deprecation` + `Sunset` +
  `Warning` (non-blocking — the skill keeps working but surfaces the notice).
- **Version below floor, past sunset** → the server **fails closed before any state
  change** (no quote honored, no trade submitted):

```
HTTP/1.1 426 Upgrade Required
```
```json
{
  "error": "skill_version_unsupported",
  "min_skill_version": "1.3.0",
  "api_revision": "2026-09-01",
  "upgrade": "Update the Treasures skills — https://github.com/treasures-io/treasures-finance-agent-skills"
}
```

**A request with no skill header is never gated this way** — no `426
skill_version_unsupported`, no skill `Deprecation`. It is a generic client.

`426 Upgrade Required` is deliberate — unambiguous and not otherwise in the API's
error namespace. The `upgrade` string is **server-controlled** so the fix instruction
lives in one place; skills relay it verbatim rather than hardcoding it.

> The `upgrade` value is runtime-agnostic on purpose (skills run on many agent
> runtimes). Point it at update guidance that covers both channels — Claude Code
> (`/plugin marketplace update`) and a re-fetch of the skills for any other runtime
> (`npx skills add treasures-io/treasures-finance-agent-skills`, per the README).

## Deprecation window policy (the opt-in gate)

When a change would break older skills, on the **skill-gate** track:

1. Bump `X-Treasures-Api-Revision`.
2. Raise the **recommended** skill version, but keep `X-Min-Skill-Version` at the old
   floor for **≥ 60 days**.
3. During that window, emit `Deprecation` + `Sunset` + `Warning` to opted-in callers
   below the recommended version (they keep working — non-blocking).
4. After the window, raise `X-Min-Skill-Version` → opted-in callers below it now get
   `426`.

This gives skill users a visible, dated runway, while guaranteeing they cannot keep
trading on a removed contract past the deadline.

## Standard versioning (the universal safety net)

This track covers **all** clients and carries the real "can't silently break"
guarantee — it does not depend on any skill header:

- The contract is pinned by the path version (`/api/v1`, `/public/v1`). A **breaking**
  change ships a new path version; the old one keeps working through a `Sunset`
  window, then returns `410 Gone`.
- A stale client hitting a changed/retired contract receives ordinary signals every
  client respects: `Sunset` on the versioned path, `410 Gone` after retirement, `404`
  for a removed endpoint, or `400 invalid_request` for a body that no longer validates.
- Skills react to these too (see Skill behavior) — so drift surfaces even for a skill
  call that happened to omit the header.

## Skill behavior (both skills implement)

Each skill's `SKILL.md` is the normative, agent-facing home for these rules (this doc is
the spec they implement). In brief: **send** the two skill headers (the fetch helper
does it, opting into the gate); `Deprecation`/`Sunset`/`Warning` → **non-blocking** (warn
the user with the `Sunset` date + `upgrade`/`Link`, keep working); `426
skill_version_unsupported` → **blocking** (stop, do not retry, relay `upgrade`);
standard-versioning signals (`Sunset` on the versioned path, an unexpected `410`/`404`, or
a new `400 invalid_request` on a previously valid body) → treat as possible staleness.

## Placeholders to set

Template values a real deployment still fills in:

- **`api_revision`** — your real current contract date (e.g. `2026-09-01`).
- **`upgrade` string** — the runtime-agnostic update guidance above (server-controlled).

This spec lives in the public repo (`docs/skill-version-compatibility.md`); there's no
docs-site skill page yet. The `SKILL.md` entry docs don't link here — they carry the operative
rules inline — so the only outward link that matters is the server's `upgrade` string, which
should point users at update guidance (the repo README, or a docs-site page once one exists).
