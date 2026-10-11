# Test evidence

Repository: https://github.com/Bharath-Maran/n8n-sprint2-issue-triage
Workflow: *Sprint 2 - GitHub Issue Triage* (n8n at `http://localhost:5678`, GitHub CLI forwarding running)

Every required issue is checked for four things: one n8n execution, one `github_issue_ledger` row, one `github_issue_route_log` row with the expected route name, and ledger `route_log_status = recorded`.

## Tests 1 to 4: the four issue types

| Test | Issue title (exact) | Expected category / route | GitHub issue | n8n execution | Ledger row | Route-log row |
|---|---|---|---|---|---|---|
| 1 | Bug: blank screen after logout | bug / `bug-triage` | `issue_key` `Bharath-Maran/n8n-sprint2-issue-triage#30` | ![Issue 1 - Execution](image-1.png) | ![Issue 1 - Ledger Row](image-2.png)| ![Issue 1 - Route-log Row](image-3.png) |
| 2 | Feature request: export task list to CSV | feature / `feature-requests` | `issue_key` `Bharath-Maran/n8n-sprint2-issue-triage#31` | ![Issue 2 - Execution](image-5.png) | ![Issue 2 - Ledger Row](image-6.png) |  ![Issue 2 - Route-log Row](image-7.png) |
| 3 | Help: change the public webhook address | help / `support` | `issue_key` `Bharath-Maran/n8n-sprint2-issue-triage#32` | ![Issue 3 - Execution](image-9.png) | ![Issue 3 - Ledger Row](image-10.png) | ![Issue 3 - Route-log Row](image-11.png) |
| 4 | Documentation: clarify local setup | other / `general-triage` | `issue_key` `Bharath-Maran/n8n-sprint2-issue-triage#33` | ![Issue 4 - Execution](image-13.png) | ![Issue 4 - Ledger Row](image-14.png) | ![Issue 4 - Route-log Row](image-15.png) |

**Test 1 delivery proof.** GitHub's Recent Deliveries shows the live `issues.opened` delivery `0fe96cba-c4d2-11f1-89c1-41f554098d0c` answered `200` with body `{"message":"Workflow was started"}`. That body is n8n's own immediate-response text, so the event reached the local Webhook node through the CLI forwarder.

## Redelivery test (Test 1, same delivery)

**Required result:** the original `issue_key` stays at exactly one ledger row (`last_seen_at` may change) and there is no second `bug-triage` route-log row.

### A. Native GitHub "Redeliver" (attempted, blocked)

I used **Settings → Webhooks → [webhook] → Recent deliveries → [delivery] → Redeliver** while `gh webhook forward` was running. Every attempt returned `401 Unauthorized` from `https://webhook-forwarder.github.com/hook`, GitHub's relay, with a 13-byte plain-text body `Unauthorized` and no n8n response headers. The redelivered request therefore never reached n8n.

| Time (local, 2026-10-10 unless noted) | Issue / delivery | Temporary hook ID | Original delivery | Redelivery |
|---|---|---|---|---|
| 13:22:07 and 13:26:29 | #30 / `7de70176-c5a1-11f1-8cba-907ca9435f96` | 695343734 | **200** `{"message":"Workflow was started"}` | **401** |
| about 13:39 | #31 / `177c3a26-c5a3-11f1-8765-2f00d5fe49bf` | 695352636 | **200** `{"message":"Workflow was started"}` | **401** |
| 13:43:12, 13:43:30, 13:57:03 | #32 / `60b04264-c5a3-11f1-9319-b5215b9b3342` | 695354375 | **200** `{"message":"Workflow was started"}` | **401** |
| 2026-10-11 14:31 | #33 / `b664219e-c5a3-11f1-941a-5e16588214a5` | 695352645 | **200** `{"message":"Workflow was started"}` | **401** |

What I ruled out:

- Not an n8n or workflow problem: the 401 response comes from the relay edge (`X-Github-Backend: Kubernetes`) and the same workflow returned 200 for the live deliveries.
- Not a stopped forwarder in the final attempt: the forwarder terminal was running when the 2026-10-11 redelivery failed.
- Listing deliveries through the API failed with 404 for hook 695352636 even after `gh auth refresh -s admin:repo_hook`. GitHub's hook page states the CLI-created hook goes away when the client disconnects, which is consistent with that hook no longer existing, but I did not verify this.

I do not know why the relay rejects redelivery, and I did not find documentation that `gh webhook forward` supports Redeliver. I treat it as unsupported for this setup.

![](image-17.png)

### B. Equivalent test: replay the same payload twice

Because native Redeliver was blocked, I replayed the **exact captured Test 1 payload** (the `body` object from the Test 1 execution input, saved as `payload.json`) directly to the local production webhook. This sends the same event through the same duplicate guard.

```
curl -s -X POST http://localhost:5678/webhook/sprint2-issue-triage -H "Content-Type: application/json" -d @payload.json
curl -s -X POST http://localhost:5678/webhook/sprint2-issue-triage -H "Content-Type: application/json" -d @payload.json
```

| Check | Before replay | After replay | Evidence |
|---|---|---|---|
| Ledger rows for `Bharath-Maran/n8n-sprint2-issue-triage#28` | 1 | 1 | ![Ledger Rows](image-22.png) |
| Ledger `last_seen_at` | 2026-10-11 14:28:02 | 2026-10-11 14:35:23 | Values shown in this table |
| `bug-triage` route-log rows for that key | 1 | 1 | ![Route-log Rows](image-21.png) |
| Executions | 1 | 2 | Execution list in n8n |

Limitation: this proves the duplicate guard, not that GitHub's Redeliver button works with CLI forwarding.

## Opened-action gate test

After Tests 1 to 4, I appended `Edited only to test that non-opened issue actions do not enter triage.` to the body of the Test 4 issue.

| Expected | Evidence |
|---|---|
| GitHub sends an `issues.edited` delivery; n8n execution stops on the false output of **Opened action?**; no new ledger row or route-log row | Ledger and route-log contents in **Final table state** below |

Supporting observation from the build session (2026-10-07): GitHub `ping` events reached **Opened action?** and left on the false branch with no ledger write.

## Final table state
![Github Issue Ledger](image-18.png)
![Github Issue Route Log](image-19.png)
