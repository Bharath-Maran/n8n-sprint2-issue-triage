# Runbook: GitHub Issue Triage (Sprint 2)

Workflow: *Sprint 2 - GitHub Issue Triage* in local n8n (`http://localhost:5678`). Repository: https://github.com/Bharath-Maran/n8n-sprint2-issue-triage

## Purpose

Turn each newly opened issue in the test repository into one triage ledger row and one internal route-log row, and make sure a repeated delivery or a route-log failure never loses or duplicates a record. This is a local prototype run only while the GitHub CLI forwarder is open.

## Normal operation

1. In n8n, open the workflow and confirm the **Active** toggle is on.
2. In a separate terminal, start forwarding and leave it open:
   ```
   gh webhook forward --repo=Bharath-Maran/n8n-sprint2-issue-triage --events=issues --url=http://localhost:5678/webhook/sprint2-issue-triage
   ```
   Expected: the terminal prints `Forwarding Webhook events from GitHub...` and then waits with no prompt. If it exits or prints an error, fix that before creating any issue.
3. Create a test issue in the repository.
4. Open the newest execution in n8n. Expected: **Opened action?** takes the true output, one category branch runs, **Insert pending ledger row** and **Record internal route** run, and **Update route-log recorded** runs.
5. Open the Data Tables. Expected: one new `github_issue_ledger` row with `route_log_status = recorded` and one new `github_issue_route_log` row whose `route_name` matches the category (`bug-triage`, `feature-requests`, `support`, or `general-triage`).

Evidence: `test_evidence.md`, Tests 1 to 4.

## Duplicate delivery

**What it is:** GitHub (or a manual replay) delivers the same `issues.opened` event again. The `issue_key` (`owner/repo#number`) already exists in the ledger.

**What happens:** **If ledger row exists** passes the item to **Update existing last seen**, which changes only `last_seen_at`. **If ledger row does not exist** passes nothing, so no ledger insert and no route-log write occur.

**How to check:**
1. Open the newest execution. Expected: the exists branch ran and **Insert pending ledger row** did not.
2. In `github_issue_ledger`, filter by the `issue_key`. Expected: one row, with a newer `last_seen_at`.
3. In `github_issue_route_log`, filter by the `issue_key`. Expected: still one row.

**To test a duplicate on purpose:** GitHub's Recent Deliveries **Redeliver** returned `401` from the CLI relay in this setup, so replay the captured payload instead:
```
curl -s -X POST http://localhost:5678/webhook/sprint2-issue-triage -H "Content-Type: application/json" -d @payload.json
```
where `payload.json` is the `body` object from the original execution's input. Expected response: `{"message":"Workflow was started"}`.

Evidence: `test_evidence.md`, redelivery section.

## Route-log failure

**What it is:** the **Record internal route** write fails (or, in the course test, **Simulate route-log failure** throws on a title containing `intentional route-log failure test`).

**What happens:** the ledger row was already inserted as `pending`. The error output runs **Update route-log failed**, which sets `route_log_status = failed` and `error_note = "Route-log write failed; review the route-log node configuration and execution details."`. No route-log row is written. The event is never silently dropped.

**How to detect it:**
1. In `github_issue_ledger`, filter `route_log_status` equals `failed`.
2. In n8n **Executions**, filter by failed or open the matching execution to see which node errored.

Evidence: `test_evidence.md`, Test 5.

## Recovery action

A failed row is **not** retried by sending the event again, because the duplicate guard sees the existing `issue_key` and only updates `last_seen_at`. Recover in one of two ways, after fixing the cause (for example a Data Table schema or node configuration problem shown in the execution details):

**Option A: re-run the event**
1. In `github_issue_ledger`, delete the failed row for that `issue_key`.
2. Replay the original payload with the `curl` command above.
3. Expected: a new `pending` row, a route-log row, then `route_log_status = recorded`.

**Option B: repair in place**
1. In `github_issue_route_log`, add one row by hand with the ledger row's `issue_key`, `category`, `route_name`, and the current time as `recorded_at`.
2. In `github_issue_ledger`, set `route_log_status` to `recorded` and clear `error_note`.

Use Option B when the original payload is no longer available. After either option, confirm exactly one ledger row and one route-log row for the `issue_key`.

For the deliberate Test 5 row, no recovery is needed. It is left as `failed` as evidence.

## Safe shutdown

After all evidence is captured:

1. Export the workflow (kept privately) and save the documentation.
2. In n8n, toggle the workflow to **Inactive**.
3. In the forwarder terminal press **Ctrl+C**. The CLI-created webhook is removed when the client disconnects.
4. In GitHub, open **Settings → Webhooks** and confirm no webhook remains for this repository. If one remains, delete it.
5. Optional: remove the extra CLI permission added while troubleshooting: `gh auth refresh -h github.com -r admin:repo_hook`.
6. Do not leave test issues open that could be mistaken for real work; the repository holds only synthetic test content.

## What this prototype does not test

- **Production hosting:** n8n runs on `localhost`, and events arrive only while `gh webhook forward` is running. Events created while it is stopped are not received.
- **Native GitHub redelivery:** blocked in this setup (`401` from the relay). Duplicate handling was tested by replaying the same payload.
- **Concurrent duplicates:** the duplicate guard is two separate "exists / does not exist" checks, not an atomic database constraint. Two identical deliveries arriving at the same instant could both pass the "does not exist" check. This was not tested.
- **Webhook authenticity:** the webhook has no secret and the workflow does not verify a signature.
- **Retry or alerting:** failed rows are visible only if someone looks; there is no automatic retry, notification, or timer.
- **Event types:** only `opened` issue events are handled; labels, comments, and pull requests are out of scope.
- **Volume and long-term storage:** n8n Data Tables are not tested for high volume or backup.
