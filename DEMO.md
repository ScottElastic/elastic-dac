# DAC Demo Runbook

**Runtime:** ~8 minutes  
**Audience:** Security practitioners, SecOps managers

---

## Pre-demo setup (5 min before)

- [ ] Terminal open, cloned repo (`cd elastic-dac`)
- [ ] GitHub repo open: `github.com/ScottElastic/elastic-dac`
- [ ] Kibana open and logged in — keep this tab ready for the Act 1 payoff
- [ ] Files staged: `demo/` folder visible in your editor

---

## Opening hook (30 sec)

> *"Every security team I talk to manages detection rules the same way — someone logs into Kibana,
> clicks around, saves something. Maybe they test it, maybe they don't. There's no review, no history,
> no rollback. When a rule goes noisy at 2am, nobody knows who wrote it or why.*
>
> *DAC treats your detection rules exactly like application code. Let me show you what that looks like."*

---

## Act 1 — New threat intel → rule live in Kibana in under 60 seconds

**Set the scene (30 sec)**

Open `demo/threat-intel-brief.md`.

> *"GhostLoader ransomware just hit Little Pharma — it was on their leak site two days ago.
> Incident responders found a very specific two-step sequence on every compromised host:
> bcdedit disables Windows recovery, then vssadmin wipes shadow copies. We need a rule for this now."*

**Write and deploy the rule (2 min)**

```bash
# Copy the pre-written rule into the rules directory
cp demo/act1-ghostloader-ransomware-staging.toml custom-rules/rules/

# Branch, commit, push
git checkout -b feat/ghostloader-ransomware-staging
git add custom-rules/rules/act1-ghostloader-ransomware-staging.toml
git commit -m "feat: detect GhostLoader ransomware staging (bcdedit + VSS deletion)"
git push -u origin feat/ghostloader-ransomware-staging

# Open a PR
gh pr create --fill
```

**Walk the PR (1 min)**

- Open the PR on GitHub — point out the PR template auto-populated
- Watch **Validate Rules (PR)** kick off → it passes ✅
- Merge the PR
- Watch **Deploy Rules to Kibana** trigger automatically

**Switch to Kibana tab** — the rule is live in Security → Rules.

> *"Threat intel in. Rule deployed. Full audit trail. Nobody touched Kibana directly."*

---

## Act 2 — The bad rule gets blocked (2 min)

**Set the scene (15 sec)**

> *"Now let's say an analyst writes a new rule but makes a mistake — sets the severity to
> something that doesn't exist in the schema. In Kibana, you'd save it fine and find out
> when it breaks at runtime. Here, the pipeline catches it before it ever gets close to production."*

**Push the bad rule**

```bash
cp demo/act2-bad-eql-rule.toml custom-rules/rules/

git checkout -b feat/process-hollowing-rule
git add custom-rules/rules/act2-bad-eql-rule.toml
git commit -m "feat: detect process hollowing via remote thread injection"
git push -u origin feat/process-hollowing-rule

gh pr create --fill
```

**Watch it fail (30 sec)**

- Open the PR on GitHub
- Watch **Validate Rules (PR)** run → it fails ❌
- Click into the failed step → show the error: `invalid value for severity: 'super-critical'`
- The **Merge** button is greyed out — GitHub blocks it

> *"The pipeline is the gatekeeper. Fix the rule in code, push again, CI re-runs.
> Nothing reaches production until it's valid."*

---

## Closing (1 min)

| Old world | With DAC |
|---|---|
| Rules live only in Kibana | Git is the source of truth |
| No review process | Every change is a PR, reviewed and approved |
| No history — who changed what? | `git log` and `git blame` on every rule |
| Bad rule in prod until someone notices | CI blocks invalid rules before merge |
| Rollback = manual Kibana edits | `git revert` + auto-deploy |
| UI edits go undetected | Nightly sync catches drift, opens a PR |

> *"One more thing — if someone does go rogue and edits a rule directly in Kibana,
> the nightly sync job exports it, diffs it against the repo, and opens a PR automatically.
> Nothing stays hidden."*

---

## Cleanup after demo

```bash
# Close/delete the bad rule branch — it was never merged
gh pr close <PR number> --delete-branch

# Optionally remove the Act 1 rule if you don't want it in the persistent repo
git checkout main
git rm custom-rules/rules/act1-ghostloader-ransomware-staging.toml
git commit -m "chore: remove demo rule post-presentation"
git push
```
