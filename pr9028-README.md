# PR #9028 — wire-level capture (rule-title username scrub)

Captured on the integration build `deploy/vm-v35` (`d2844d3b0`, all nine wave PRs merged) running
on the packaged 5.0 dashboard in the `wazuh-aio-5` VM, 2026-08-24.

Method (the established one from `AI/qa/PRIVACY-CAPTURE-HANDOFF.md`): a local
OpenAI-compatible endpoint records the COMPLETE outbound request bodies to JSONL; a scratch
provider is POSTed at it and DELETEd afterwards, so no existing provider is touched. Privacy mode
on for every turn.

Two probes, both against the VM's real seeded findings:

- **PROBE-A** — `search_wazuh_data` projecting only `source.user.name` (the 20 rows whose account
  is `vagrant`), so the pseudonymizer mints `USER_1` from a real `anonymize`/`USER` field; then
  `get_top_rules`, whose digest samples carry
  `wazuh.rule.title = "Successful user authentication - vagrant"`.
- **PROBE-B** — the same two calls in the **reverse** order, so the title digest is serialized
  while `vagrant` is still unknown.

## Result

PROBE-A, final outbound request of the turn — the title arrives pseudonymized:

```
{"wazuh.rule.title": "Successful user authentication - USER_1", "doc_count": 2344}
{"wazuh.rule.title": "Wazuh VD - Vulnerability CVE-2026-43492 remediated", "doc_count": 796}
{"wazuh.rule.title": "Failed authentication attempt - auditbot from IP_1", "doc_count": 79}
{"wazuh.rule.title": "Sudo command executed - USER_1", "doc_count": 34}
{"wazuh.rule.title": "syslog: User authentication failure", "doc_count": 32}
```

PROBE-B, request 7 — the digest as first serialized, before the mint (this is the form that would
be **persisted**, which is what `premintProseScanIdentifiers` exists to fix):

```
{"wazuh.rule.title": "Successful user authentication - vagrant", "doc_count": 2344}
{"wazuh.rule.title": "Sudo command executed - vagrant", "doc_count": 34}
```

PROBE-B, request 8 — the same digest replayed after the mint, scrubbed by the end-of-turn
outbound pass:

```
{"wazuh.rule.title": "Successful user authentication - USER_1", "doc_count": 2344}
{"wazuh.rule.title": "Sudo command executed - USER_1", "doc_count": 34}
```

Across the whole capture (8 outbound requests) the literal string `vagrant` appears **twice**,
both inside PROBE-B's deliberately pre-mint request 7.

Two of the PR's documented residuals are visible in the same lines, unchanged:

- `auditbot` stays verbatim — it is dotless and appears in no pseudonymized field anywhere in the
  conversation, so there is no pseudonym to reuse (first residual in the PR body).
- The IP inside the same title is `IP_1` — the pre-existing value-shape scan, not this change.

Full capture: `pr9028-wire-capture.jsonl` (8 records, complete request bodies, auth headers
redacted).
