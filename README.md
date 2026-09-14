# echo-earning-agent

An **always-on autonomous earning agent** that runs on GitHub Actions — i.e. on GitHub's
servers, on a schedule, **whether or not any home computer is on**. No new accounts: it uses
the GitHub identity we already have.

Every 30 minutes it:
1. reads the on-chain balance of our receive-only wallet(s) — real earnings show up in
   [`status.md`](status.md);
2. scans [Superteam](https://superteam.fun)'s agent-listing API for new/open bounties worth
   entering, flagging anything new since the last run;
3. commits a fresh `status.md` + appends `history.jsonl`, so progress is visible any time you
   glance at the repo — no terminal, no local process.

**Safety:** this repo holds **no private keys**. The agent only ever *reads* public chain data
and public listings. Anything that spends or signs stays offline on the operator's machine.

## Setup

Repo files are already enough for the agent to run. Remaining work is GitHub (and optionally
Deno) settings — nothing to install locally.

`status.md` / `history.jsonl` on a fresh fork are a **snapshot copied from upstream**. They
are not proof that *this* repo's Actions are running. Confirm a run under **this** repo's
Actions tab.

### 1. Enable GitHub Actions (required on forks)

GitHub **does not run workflows on a fork until you enable them**, and **scheduled workflows
stay off on forks by default**.

1. Open this repo's **Actions** tab.
2. If you see a banner that workflows are not being run on this fork, click
   **I understand my workflows, go ahead and enable them**.
3. Open the **earning-agent** workflow and use **Enable workflow** if it is disabled.
4. **Settings → Actions → General**:
   - Allow Actions (all actions, or at least the official `actions/*` ones).
   - **Workflow permissions:** read and write (needed so the job can push `status.md`).
     The workflow also sets `permissions: contents: write` in YAML.

Until this is done, cron will never fire on this repo even though `.github/workflows/agent.yml`
is present.

### 2. Secrets (all optional)

**Settings → Secrets and variables → Actions → New repository secret.**

Forks **do not inherit** secrets. If you forked this, you must re-add them here.

| Secret | Required? | What it unlocks |
| --- | --- | --- |
| `SUPERTEAM_API_KEY` | optional | Superteam live listings + Imperial hackathon watch |
| `DEALWORK_API_KEY` | optional | dealwork.ai heartbeat, bids, and escrow contracts |
| `TOKU_API_KEY` | optional | toku.agency wallet / unread notifications |
| `GITHUB_TOKEN` | automatic | Provided by Actions; used to search authored PRs |

Without a given key, that rail is skipped (`_scan skipped: no … secret_` in `status.md`).
The wallet reads, OpenTask probe, and x402 health checks still run.

**Do not** put API keys, `.env` files, or wallet private keys in the repo.

### 3. Email on earning events

On a real event (payment, winners, escrow, merged PR) the agent writes `NOTIFY.txt` and the
workflow **fails on purpose** so GitHub can email you. That file is local to the job (gitignored).

1. GitHub **Settings → Notifications → Actions** → email on failed workflows.
2. Watch this repository (Participating, or All Activity if you want every run).

A red **earning-agent** run with `🏆 EARNING EVENT` is the alert — not a broken agent.

### 4. Verify a run

1. **Actions → earning-agent → Run workflow** (manual `workflow_dispatch`).
2. The job should finish in about a minute.
3. `status.md` **Last run** should be a fresh UTC timestamp, and `main` should gain a
   `status @ …` commit from `echo-earning-agent`.
4. Cron is `13,43 * * * *` (every 30 minutes). GitHub may delay public-repo schedules
   by hours under load; a delayed run is still a run.

Local dry-run (does not need secrets; **do not commit** the rewritten status files):

```bash
node agent.mjs
```

Requires Node 20+.

### 5. Paid x402 service (not this repo)

[`https://token-intel-x402.echolonius.deno.net`](https://token-intel-x402.echolonius.deno.net)
is a **separate Deno Deploy** app. Its source is not in this repository. The agent only
probes `/healthz`, the paid route, and `/api/token-intel/demo`.

A `503` with `USAGE_EXCEEDED` means the Deno project is suspended on usage limits — fix that
in the Deno dashboard (raise limits / redeploy), not by changing this agent.

## Roadmap (making money land while you're away)
- [ ] Host the x402 paid service off-box (serverless) so it sells even when the home box is off
- [ ] Auto-refresh the service's discovery listings so buyers can always reach it
- [ ] Auto-draft + submit to fitting agent bounties (quality-gated, never spam)
- [ ] Notify on new high-value listings
