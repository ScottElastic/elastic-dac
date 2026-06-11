# Elastic Security — Detections as Code

Custom detection rules managed as code using the official [elastic/detection-rules](https://github.com/elastic/detection-rules) CLI, following the [DaC Reference](https://dac-reference.readthedocs.io/) end-to-end example.

**Governance model:** VCS-authoritative with dual sync. `main` is the source of truth; a nightly export job catches anything edited in the Kibana UI and surfaces it as a PR.

## Repo layout

```
.
├── .github/workflows/
│   ├── pr-validation.yml      # schema + query validation + unit tests on every PR
│   ├── deploy-to-kibana.yml   # imports rules to Kibana on merge to main
│   └── sync-from-kibana.yml   # nightly export → PR if drift is detected
└── custom-rules/              # CUSTOM_RULES_DIR
    ├── _config.yaml           # rule loader config (bypass_version_lock, etc.)
    ├── rules/                 # custom rule TOML files
    ├── exceptions/            # exception lists
    ├── actions/               # rule actions
    ├── action_connectors/     # action connectors
    └── etc/                   # test config, stack-schema-map, version lock
```

Prebuilt Elastic rules are intentionally **not** managed here — they're versioned and updated by Elastic via the prebuilt rules package. Only custom rules live in this repo.

## One-time setup

### 1. Create a Kibana API key

In Kibana: **Stack Management → API Keys → Create API key**. It needs read/write on the Security Solution rules APIs (the `manage_security` cluster privilege or an appropriately scoped role).

### 2. Configure GitHub secrets and variables

| Name | Type | Value |
|---|---|---|
| `KIBANA_URL` | secret | e.g. `https://<project>.kb.us-east-1.aws.elastic.cloud` |
| `KIBANA_API_KEY` | secret | the encoded API key |
| `KIBANA_SPACE` | variable (optional) | defaults to `default` |

Also create a `production` GitHub environment (Settings → Environments) — the deploy job is gated on it, so you can require manual approval before rules hit Kibana.

### 3. Initial export (seed the repo with everything in Kibana)

Three starter rules are already in `custom-rules/rules/`. To pull **all** custom rules with full fidelity, run locally:

```bash
git clone https://github.com/elastic/detection-rules.git
cd detection-rules && pip install ".[dev]"
export CUSTOM_RULES_DIR=/path/to/this-repo/custom-rules

python -m detection_rules kibana \
  --kibana-url "$KIBANA_URL" --api-key "$KIBANA_API_KEY" \
  export-rules -d "$CUSTOM_RULES_DIR/rules" \
  -ed "$CUSTOM_RULES_DIR/exceptions" \
  --custom-rules-only --export-exceptions --skip-errors
```

(Or just trigger the **Sync Rules from Kibana** workflow from the Actions tab once secrets are set — it does the same thing and opens a PR.)

Note: the export will overwrite the three starter TOMLs with the exact versions from Kibana (matched by `rule_id`) — that's expected and correct.

## Day-to-day workflow

1. Branch, add/edit a TOML in `custom-rules/rules/` (or use `python -m detection_rules create-rule` interactively).
2. Validate locally: `python -m detection_rules validate-all` (run from the detection-rules checkout with `CUSTOM_RULES_DIR` set).
3. Open a PR → validation + unit tests run automatically.
4. Merge → rules deploy to Kibana automatically (with environment approval if configured).

## Useful CLI commands

```bash
# Validate everything
python -m detection_rules validate-all

# Run unit tests (scoped to custom rules via etc/test_config.yaml)
python -m detection_rules test

# Pretty-print a rule as the API payload Kibana will receive
python -m detection_rules view-rule custom-rules/rules/<file>.toml

# Interactive rule builder
python -m detection_rules create-rule custom-rules/rules/<new>.toml

# Import a single rule manually
python -m detection_rules kibana --kibana-url ... --api-key ... \
  import-rules -f custom-rules/rules/<file>.toml --overwrite
```

## Config notes

`custom-rules/_config.yaml` is set up per the DaC reference recommendations:

- `bypass_version_lock: True` — versioning is delegated to Kibana's revision field rather than the repo's `version.lock.json` (simplest model for custom rules).
- `normalize_kql_keywords: True` — consistent casing means clean diffs between exports.
- `auto_gen_schema_file` — non-ECS/custom fields (e.g. CrowdStrike FDR fields) get auto-added to a generated schema instead of failing validation.
- `bypass_optional_elastic_validation: True` — skips Elastic-internal requirements (investigation guides, license headers) that don't apply to custom rules.

`custom-rules/etc/test_config.yaml` bypasses the Elastic-internal unit tests that only make sense for the prebuilt ruleset.
