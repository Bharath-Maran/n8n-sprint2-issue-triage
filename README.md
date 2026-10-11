# Sprint 2: GitHub Issue Triage Automation (n8n)

**Public repository:** https://github.com/Bharath-Maran/n8n-sprint2-issue-triage (visibility: **Public**)


## Purpose

A self-hosted n8n workflow that receives **newly opened GitHub issues** from this public test repository, classifies each one from its title with deterministic rules (no LLM), saves a durable triage record in an n8n Data Table, writes a category-specific entry to an internal route log, and handles a duplicate webhook delivery without creating a duplicate ledger row or route-log row.

## Services

| Layer | What I used | Role |
|---|---|---|
| Event source | This public GitHub repository + GitHub CLI `gh webhook forward` | Emits `issues` events and forwards them to localhost while the command runs |
| Automation host | Self-hosted n8n at `http://localhost:5678` | Receives, filters, classifies, routes |
| State and evidence | Two n8n Data Tables: `github_issue_ledger`, `github_issue_route_log` | Durable triage record and route outcome |

No external notification service, credential, tunnel account, or public domain is used. This is a local development/testing setup, not a production deployment.

## Node sequence

Workflow name: **Sprint 2 - GitHub Issue Triage**

1. **GitHub issue webhook** (Webhook, POST, path `sprint2-issue-triage`, respond immediately)
2. **Opened action?** (IF: `{{ $json.body.action }}` equals `opened`). Only the true output continues, so edits, closes, reopens and GitHub `ping` events stop here with no ledger or route-log write.
3. **Normalize issue** (Edit Fields: the short, stable fields listed under Field mapping)
4. **Classify title** (Switch, rules mode, first match wins: bug, feature, help, fallback other)
5. **Set bug route / Set feature route / Set help route / Set other route** (Edit Fields: fixed `category` and `route_name`)
6. Duplicate guard: each route node feeds **two parallel checks** on the same `issue_key`
   - **If ledger row exists** then **Update existing last seen** (updates only `last_seen_at`; stops there)
   - **If ledger row does not exist** then **Insert pending ledger row** (`route_log_status = pending`)
7. **Intentional failure test?** (IF: `title_lower` contains `intentional route-log failure test`)
   - false: **Record internal route** (inserts a `github_issue_route_log` row) then **Update route-log recorded** (ledger `route_log_status = recorded`, blank `error_note`)
   - true: **Simulate route-log failure** (Code node that throws)
   - error output of **Record internal route** and of **Simulate route-log failure**: **Update route-log failed** (ledger `route_log_status = failed`, safe fixed `error_note`)

The pending ledger row is written **before** the route-log write, so a route-log failure still leaves a durable record that the event arrived.

## Field mapping

`Normalize issue` maps these fields from the webhook payload (they sit under `body` in n8n):

| New field | Expression |
|---|---|
| `repository_full_name` | `{{ $json.body.repository.full_name }}` |
| `issue_number` | `{{ $json.body.issue.number }}` |
| `issue_key` | `{{ $json.body.repository.full_name + "#" + $json.body.issue.number }}` |
| `issue_url` | `{{ $json.body.issue.html_url }}` |
| `title` | `{{ $json.body.issue.title }}` |
| `title_lower` | `{{ $json.body.issue.title.toLowerCase() }}` |
| `reporter` | `{{ $json.body.issue.user.login }}` |
| `opened_at` | `{{ $json.body.issue.created_at }}` |
| `last_seen_at` | `{{ $now.toISO() }}` |

**Ledger table `github_issue_ledger`:** `issue_key`, `issue_url`, `title`, `category`, `reporter`, `opened_at`, `last_seen_at`, `route_log_status` (pending / recorded / failed), `route_name`, `error_note`.
**Route-log table `github_issue_route_log`:** `issue_key`, `category`, `route_name`, `recorded_at`.

`issue_key` (`owner/repo#number`) is the idempotency key: the same GitHub issue always produces the same key.

## Exact classification rules

The title is lower-cased, then the first matching rule wins:

| Order | Rule (regex on lower-cased title) | Category | Route name |
|---|---|---|---|
| 1 | `\b(bug\|error)\b` | bug | `bug-triage` |
| 2 | `\b(feature\|request)\b` | feature | `feature-requests` |
| 3 | `\b(help\|question)\b` | help | `support` |
| 4 | anything else (fallback) | other | `general-triage` |

Because the first match wins, a title containing both "bug" and "request" is a bug. Whole-word matching means "debugging" does not match `bug`.

## Setup notes

1. Create the public repo `n8n-sprint2-issue-triage` with a README; confirm the Issues tab exists.
2. In n8n, create the two Data Tables with the columns above, build the workflow, and set **Active**.
3. `gh auth login`, then `gh extension install cli/gh-webhook`.
4. In a dedicated terminal, start forwarding and **leave it running** for the whole test session:
   ```
   gh webhook forward --repo=Bharath-Maran/n8n-sprint2-issue-triage --events=issues --url=http://localhost:5678/webhook/sprint2-issue-triage
   ```
5. Create the test issues one at a time and verify each in n8n before creating the next.

`gh webhook forward` creates a temporary repository webhook that GitHub says "will go away when the client disconnects". Redelivering a webhook from GitHub's Recent Deliveries returned `401 Unauthorized` from `webhook-forwarder.github.com` in every attempt (see `test_evidence.md`), so the duplicate-delivery test was completed by replaying the captured payload directly to the local webhook.

## Evidence

- Per-test evidence, redelivery result, and failure test: [`test_evidence.md`](test_evidence.md)
- Operating and recovery instructions: [`runbook.md`](runbook.md)
- Self-hosted vs Zapier comparison: [`own_vs_rent.md`](own_vs_rent.md)
- Workflow export (credentials redacted): [`sprint2_github_issue_triage.json`](sprint2_github_issue_triage.json)
- Week 6 journal: [`journals/week-06.md`](journals/week-06.md)

## Safety

This repository contains only the course's harmless test issues. No credentials, tokens, webhook secrets, or personal data are committed. Screenshots of n8n webhook headers are cropped so request signatures are not shown.
