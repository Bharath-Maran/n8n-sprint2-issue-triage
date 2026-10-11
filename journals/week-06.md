# Weekly Build Activity Journal

## CS 590 Low Code No Code for Rapid Prototyping

**Student Name:** Bharathwaj Nedoumaran
**Week:** 6
**Date Range:** 10/05/2026 to 10/11/2026
**List Course Work and/or Project:** Sprint 2: GitHub Issue Triage Automation (n8n), repository https://github.com/Bharath-Maran/n8n-sprint2-issue-triage

Evidence in this journal points to files in the repository, which a reviewer can open without signing in. Screenshots are kept in `test_evidence.md` only where they are needed to show a result.

## 1. Weekly Focus and Goals [1 point]

**Objective:** Build an event-driven n8n workflow that receives new GitHub issues through GitHub CLI webhook forwarding, classifies them from the title, keeps an idempotent ledger and route log, and documents a failure path.

- Goal 1: Get opened issues from the public repository into local n8n and classify them into four categories with deterministic rules.
- Goal 2: Make the ledger duplicate-safe, so a repeated delivery does not create a second route-log row.
- Goal 3: Prove the failure path and write the README, runbook, test evidence, and Own versus Rent comparison.

## 2. Completed Work and Evidence [3 points]

| Completed work | Evidence | Why it matters to this week's objective |
|---|---|---|
| Built the workflow: webhook, opened-action gate, Normalize issue, Classify title (Switch), four route nodes, duplicate guard, route log, and failure path | `sprint2_github_issue_triage.json` and the node sequence in `README.md` | Covers the end-to-end path the sprint grades |
| Received a live GitHub issue through `gh webhook forward` | `test_evidence.md`: delivery `0fe96cba-c4d2-11f1-89c1-41f554098d0c` returned 200 `{"message":"Workflow was started"}` (issue #28) | Shows GitHub reaching local n8n with no public URL |
| Ran the four issue types, the opened-action gate test, and the route-log failure test | `test_evidence.md`, Tests 1 to 5 | Verifies classification, routing, and the failed state |
| Wrote the operating and comparison documents | `runbook.md`, `own_vs_rent.md` | Lets a reviewer operate the prototype and see the build-versus-rent reasoning |

## 3. Work in Progress and Next Step [1 point]

| Work in progress | Current state | Next concrete action |
|---|---|---|
| Native GitHub Redeliver | Blocked: every attempt returned 401 from the CLI relay | Try Redeliver on a normal webhook behind a stable public URL, or ask the instructor whether the payload replay is accepted |
| Concurrent duplicate deliveries | Not tested; only sequential duplicates were checked | Send two identical payloads at the same moment and check the ledger for one row |

## 4. What You Learned [2 points]

The clearest result was that the duplicate guard decides what recovery looks like. Because a repeated event for an existing `issue_key` only updates `last_seen_at`, sending a failed event again does not retry it. Recovery has to be deliberate: delete the failed ledger row and replay the event, or repair both tables by hand. I only noticed this while writing `runbook.md`, not while building the nodes.

I also learned to separate the webhook transport from the workflow logic. The same workflow returned 200 for live deliveries while GitHub's redelivery failed at the relay, which showed the problem sat outside my nodes.

**Prototype dimension tested most directly:** process, meaning how an event moves through filtering, duplicate checks, and failure handling, with data shape a close second.

**Question my work did not answer:** whether two identical deliveries arriving at the same moment could both pass the "does not exist" check and create two rows. The guard is two separate checks, not a database constraint, and I tested only sequential duplicates.

## 5. Challenge and Diagnosis [2 points]

- **What happened?** GitHub's Redeliver returned `401 Unauthorized` from `webhook-forwarder.github.com` on every attempt (issues #25, #27, #28, and a later retry on 2026-10-11), while the original live deliveries returned 200. The 401 response carried no n8n headers, so n8n never received the request.
- **What did I try or rule out?** I confirmed the forwarder was running during the last attempt and restarted it. I refreshed my `gh` login with the `admin:repo_hook` scope to list deliveries through the API, which returned 404 for the old hook. GitHub's hook page says the CLI webhook goes away when the client disconnects. I ruled out my workflow as the cause because live events succeed. I could not determine why the relay rejects redelivery, and I found no documentation saying it supports Redeliver.
- **What will I try next?** I tested duplicate handling by replaying the exact captured payload twice to the local webhook and checking that the ledger and route log each kept one row (`test_evidence.md`). Next I would try Redeliver on a normal webhook behind a stable URL.

## 6. Response Plan [1 point]

Next week I plan to apply what I learned about the duplicate guard to the next assignment and, if time allows, test concurrent duplicates and Redeliver on a stable URL. Instructor question: is a direct payload replay an acceptable duplicate-delivery test when Redeliver through `gh webhook forward` returns 401?

**Extensibility observation (required):** the capability I could add next without replacing the core design is a **retry and escalation sweep for failed or stuck rows**. Existing pieces make it easier: every event leaves a durable ledger row before the route-log write, `route_log_status` already has `pending`, `recorded`, and `failed`, and `issue_key` is a stable identifier a sweep could use to find and fix one row. Two design decisions make it harder. First, the duplicate path only updates `last_seen_at`, so a retry cannot simply resend the event and must reuse the stored fields. Second, `error_note` is one fixed sentence and there is no attempt count or failure timestamp, so a timer or retry limit would need a new column. The same fields (`opened_at`, `route_log_status`) would also support a service-level metric such as time from opened to recorded.

## Course Evidence Check

- [x] I included evidence that a reviewer can inspect.
- [x] I used only synthetic or course-approved data and did not include secrets, tokens, or personal information.
- [x] I named a real outcome and a real challenge rather than writing a generic status update.
- [x] If this work has a cost, security, privacy, or extensibility implication, I named it in the relevant section above.