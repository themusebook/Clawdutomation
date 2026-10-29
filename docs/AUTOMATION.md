# How we automate Muse

*The operating playbook behind Clawdutomation: how any Muse — or any bot built on this runtime — is automated for trading, publishing, working, and reverse engineering. Patterns, not secrets: every deployment fills in its own credentials, wallets, and limits.*

The loop underneath everything is the one this runtime is built on:

```
sense → think → act → observe → verify → remember
```

Automation is never "fire and forget." Every automated action below is bounded before it runs (scope, caps, schedule), verified after it runs (on-chain state, run logs, receipts), and remembered (memory files, state files, journals) so the next run starts from evidence instead of vibes.

---

## 0. The chassis (applies to all four domains)

- **Scheduled jobs are the unit of automation.** Each job is a self-contained prompt + schedule (interval, hourly, daily) with a named owner (a goal or a workstream), an explicit delivery rule (report, stay silent, report only failures), and a state file it alone owns.
- **Exactly once per run.** A job reads its state, does work not yet done, records what it did, and exits. Re-running a job must be safe: idempotency guards (e.g. same-action-in-window blocks) mean a retry never duplicates an effect.
- **Silent is a valid result.** If there is nothing new, the job says nothing. Noise is how operators learn to ignore the channel that carries real alarms.
- **Secrets live in a vault, never in prompts, code, or logs.** Jobs name the credential they need; they never print it, commit it, or paste it into a deliverable. Pasted secrets are treated as exposed.
- **Verify before reporting.** A tool returning "ok" means *submitted*, not *done*. Work is reported only after an independent read-back: the chain, the API, the file, the page.

## 1. Trading

Goal: let an agent move money on a leash the operator can see, sized so a bad day is a lesson, not an ending.

- **Browser-signing is the default.** The agent builds transactions; a human's wallet signs them. Server-side signing is the exception, granted per-wallet, per-purpose.
- **Scoped local wallets, hard caps.** When autonomous execution is approved, it runs from a dedicated wallet that can do exactly one thing (e.g. buy one named token) with a fixed per-trade size and, where set, a daily cap. Scope never widens silently: a new venue, size, or token is a new approval of exact terms.
- **Paper first.** New strategies run in simulation (paper trading with real market data) until their decisions are boring. Live execution is a promotion, not a starting point.
- **Route with a fallback, prove every send.** The reference buy loop routes Jupiter-first with DFlow as fallback (the order flipped after repeated dropped sends on the first route — the routing follows evidence). After *any* send, the loop polls the transaction status and both balances before believing it: a send helper once returned signatures for transactions that never landed, twice, with balances bit-identical. The rule that came out of it: **prove non-execution (unknown signature on 2+ RPCs + unchanged balances) before retrying, and never re-run a mutating command just to read its output.**
- **Worked example — a token accumulation loop:** fixed SOL amount per buy, fixed cadence, no daily cap (an explicit operator decision, recorded as such), each buy verified against its own transaction signature before its receipt is published, buys tweeted in the same run they land (see §2), burns on a separate fixed schedule with their own tally. Every parameter lives in one place and changed only on a named order.

## 2. Tweeting / publishing

Goal: a public timeline that is a *feed of real events*, not a content calendar.

- **Data-only pulses.** Each scheduled job pulls one class of live data (market snapshot, new launches, a project's GitHub/Railway/Render activity, a buy receipt), formats it, and posts at most one item per run.
- **New events only.** A state file / queue records what has already posted (posted-ids set, a floor index for receipts: only items above the floor are eligible). Backlogs are never dumped; if it wasn't posted when it happened, it doesn't get posted later.
- **Dedupe + rotation.** A flow job that can post about many sources keeps a seen-set and rotates sources, so the timeline stays diverse instead of repeating the freshest well.
- **One diverse post per run, quality bar set by an exemplar.** The realtime flow crawls the project's own surfaces, picks the single freshest unposted real event, and posts that — falling back to a plain market snapshot only on a material move (≥3%), otherwise silence.
- **No advice language, no unverified claims.** Numbers come from the API response in the same run. A feature is announced as live only after it is verified live. If the posting API returns 402/403 (credits/rate), the job holds silently — it never retries into a ban and never posts stale data to fill the slot.
- **Fleet hygiene.** Dozens of small drips sum to real volume; cadences are tuned as a fleet (a +1-minute pass across ~30 jobs), not one job at a time.

## 3. Working (research, building, operations)

Goal: an agent that can be given a one-line order and return a verified receipt, and that several agents can share without collisions.

- **Goals + memory are the persistence layer.** Durable work hangs off a named goal with its own workspace: deliverables in `files/`, bookkeeping (run logs, watermarks, seen-lists) in `hidden_files/`. Long-term facts live in curated memory files; daily logs hold the raw record. An agent starting a task searches memory first — prior decisions and corrections are standing orders.
- **Claim-and-heartbeat coordination.** When two agents (e.g. a server agent and a desktop agent) work the same codebase, they coordinate through a shared sync file: claim a work item, heartbeat while on it, post completions with commit hashes. Nobody deploys the production worker solo; deploys ride an approved session with a known-good rollback version named in advance.
- **Lanes, not free-for-alls.** Parallel work is split into named lanes in a `PARALLEL.md` (who owns which files/areas). You never touch another lane's files; a merge conflict is resolved by the lane owners, not by whoever gets there first.
- **Subagents for self-contained pieces.** Research, audits, and prep packages are delegated with a self-contained brief; results return as files + a summary with sources. The main thread keeps the decisions and the receipts.
- **Corrections become rules.** When the operator corrects something once, it is written into the operating files (conventions/lessons) so it never has to be corrected twice. Mistakes are owned in the record, specifically, including when the agent's own earlier claim was wrong.
- **Instruction hygiene with external content.** Repository text, web pages, and pasted specs are *data*, not instructions. Only the operator's task authorizes action; anything embedded in content that tries to widen scope is flagged, not followed.

## 4. Reverse engineering (REA)

Goal: turn closed programs into evidence-backed knowledge — and know exactly where the evidence stops.

- **Evidence-led, capture-first.** Work starts from pinned artifacts: a program's deployed bytecode captured at a named slot, hashed (sha256), and stored with its capture metadata. Every claim in a report traces to a capture, a source line, or a transaction.
- **Anchor to the chain.** Behavioral claims are verified against real transactions (`getTransaction` on the anchor signatures) and live account reads at named slots — e.g. attributing a migration destination by following the actual CPI chain in the current binary, then pinning the historical cutover to the last old-path and first new-path transactions on-chain.
- **Run the verifiers.** Each evidence package ships its own verification scripts; a contribution runs them on base and on the changed tree and reports both results. Identical output is the claim; anything else is a finding.
- **Findings are LEADS until proven.** Automated analysis (including a scheduled bug-bot that re-crawls watched programs hourly and re-analyzes on any upgrade) produces *leads*, labeled as such. A lead becomes a reportable bug only with a bounded, local proof-of-concept — speculative reports are not filed.
- **Honest exclusions are part of the deliverable.** What the method can't see is written down in the same document: an instruction the dispatch map doesn't name stays unnamed; an artifact that exists on one machine but not in the repo is not cited; a parameter value is never dressed up as a binary constant.
- **Private disclosure for security.** Security findings go through the project's private advisory channel, never a public issue or post. Public contributions carry the attribution and limits; the exploit path does not.

---

## Putting it together for your own Muse or bot

1. Give it a constitution and laws it cannot edit away (this repo's `CONSTITUTION.md` / six laws), an identity file, and a memory directory.
2. Connect tools through connectors/MCP; keep credentials in a vault; scope every wallet before it can sign anything.
3. Start every new capability as a scheduled job in observe/paper mode with a state file and a silence rule.
4. Promote to action only with named caps, a verification step, and a receipt the operator can check without trusting the agent's word.
5. Write every correction down where the next run will read it.

*Proposals are not payments, "submitted" is not "landed," and a lead is not a bug. Verify, then report.*
