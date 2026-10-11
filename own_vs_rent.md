# Own versus Rent: self-hosted n8n vs Zapier

**Scenario assumed:** 250 newly opened GitHub issues per month, about one route-log failure per month, and about one workflow change per quarter, run by a small product team with no dedicated operations person.

The n8n points come from building this sprint's workflow. The Zapier points are assumptions about a hosted tool, not measurements, and I make no pricing claims because vendor plans change.

I built the full triage workflow in self-hosted n8n: a webhook, an opened-action gate, a Switch classifier, a duplicate guard, a ledger, and a failure path. The Zapier comparison covers only the first two ideas, a GitHub "issue opened" trigger followed by a step that lower-cases the title.

Setup is where the two differ most. In n8n I installed GitHub CLI and its webhook extension, created two Data Tables, and wired about twenty nodes. A hosted tool should need only an account, a GitHub connection, and two steps, so I assume Zapier is much faster to start.

Event delivery is the biggest practical gap. My n8n instance runs on localhost and received events only while `gh webhook forward` stayed open in a terminal. Events created while it was closed would be missed, and GitHub's Redeliver button was rejected by the relay, so I tested duplicates by replaying a captured payload. I assume a hosted trigger needs no local process or open terminal.

For branching, duplicate prevention, and failure handling, n8n gave me direct control. A stable `issue_key` identified each issue, a ledger row was marked pending before the route-log write, and a failed status with a readable note stopped errors from disappearing. In Zapier I would expect to need filters or paths plus a separate storage step, within whatever limits the team's plan has.

Cost and responsibility trade against each other. At 250 issues per month n8n has no licence fee for this prototype, but costs my time: keeping it running, updating it, and checking the ledger for failed rows, because nothing alerts me. A hosted service moves uptime and updates to the vendor and adds a plan cost I would check on its current pricing page. Data location also differs: in n8n the issue text stays on my machine, while in a hosted tool it passes through the vendor, which is harmless for this public test repository but could matter for real product issues.

**Conclusion.** For a small product team with no one on call to run a server, I would rent the simple version, open an issue, format the title, and notify someone, in Zapier, since it carries no uptime burden. I would choose n8n only if the team needs the duplicate ledger, failed-status records, and route log it can inspect and own, and can host n8n on an always-on machine with a public webhook address. The conclusion would change at much higher volume or if the team had to keep issue data in its own systems.

| Dimension | Self-hosted n8n (Own) | Zapier (Rent) | Your evidence or assumption |
|---|---|---|---|
| Initial setup time | Longer: GitHub CLI, extension, two Data Tables, about twenty nodes | Shorter: account, GitHub connection, two steps | n8n from this build; Zapier assumed |
| Public webhook and credential setup | No public URL needed; `gh webhook forward` to localhost; no n8n GitHub credential | Hosted trigger; GitHub authorization through the vendor | n8n from this build; Zapier assumed |
| Ability to add branching | Switch and IF nodes with full control | Filters or paths, within plan limits | n8n from this build; Zapier assumed |
| Duplicate prevention and ledger | Data Tables keyed by `issue_key`, with a pending-then-recorded status | Would need a storage or lookup step | n8n from this build; Zapier assumed |
| Failure visibility and recovery | `failed` status and note in the ledger plus the execution log; manual recovery in the runbook | Run history and error handling provided by the vendor | n8n from this build; Zapier assumed |
| Monthly volume assumption | 250 issues per month | 250 issues per month | Assumption, not measured |
| Direct platform cost | No licence fee for this local prototype; my time is the cost | Plan fee or task limit, not verified here | n8n from this build; Zapier not checked |
| Maintenance, security, uptime responsibility | Mine: updates, restarts, keeping the forwarder running | Vendor hosts and updates the service; I manage the Zap and its permissions | Assumption |
| Data location and access | Stays on my own machine | Passes through the vendor's systems | Assumption; this repository holds only public test data |
