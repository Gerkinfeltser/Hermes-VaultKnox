# VaultKnox Local Pilot Evidence — 2026-06-26

## Scope

- Repo: `Gerkinfeltser/Hermes-VaultKnox`
- Branch: `plan/vaultknox-local-pilot`
- Starting commit: `c024f7e163d0c1ade93255ddd0a12728515f64af`
- Base security patch commit: `6ba4c6b65e210601bbd2e02c6aa2f6da98ec6c1b`
- Pilot root: `/tmp/vaultknox-pilot-oplZqF`
- Runtime dir: `/tmp/vaultknox-pilot-oplZqF/runtime`
- Real credentials used: no
- Dummy credentials used: yes

## Automated Verification — Initial Run

- Command: `/tmp/vk-fork-venv/bin/python -m pytest -q`
- Result: `310 passed in 66.61s`
- Command: `/tmp/vk-fork-venv/bin/python -m ruff check src tests`
- Result: `All checks passed!`
- Command: `bandit -q -r src/vaultknox -f json`
- Result: `bandit_results 40`, `non_low_results 0`
- Command: `pip-audit --path /tmp/vk-fork-venv/lib/python3.11/site-packages`
- Result: `No known vulnerabilities found`; local `vaultknox (0.7.0)` skipped because not on PyPI

## Isolated Pilot Environment

- Created isolated root: `/tmp/vaultknox-pilot-oplZqF`
- Created directories:
  - `/tmp/vaultknox-pilot-oplZqF/cache`
  - `/tmp/vaultknox-pilot-oplZqF/config`
  - `/tmp/vaultknox-pilot-oplZqF/data`
  - `/tmp/vaultknox-pilot-oplZqF/home`
  - `/tmp/vaultknox-pilot-oplZqF/runtime`
- No pilot command intentionally targeted `~/.vaultknox` or real credentials.

## Runtime Checks

### Autonomous Store

- Path: `/tmp/vaultknox-pilot-oplZqF/home/autonomous-smoke`
- Result:

```text
round_trip_ok True
raw_secret_printed False
files ['master.key', 'secrets.enc']
```

### Scanner Redaction

- Initial plan command `python -m vaultknox.scanner ... --json` produced an empty file because `scanner.py` has no module CLI entrypoint.
- Re-diagnosis: used source-verified API `SecretScanner(paths=[...])` plus `format_findings_json(...)`.
- Result:

```text
scanner_raw_secret_present False
scanner_finding_count 1
scanner_total_findings 1
scanner_keys ['detector_name', 'file_path', 'is_duplicate', 'line_content', 'line_number', 'secret_fingerprint', 'severity']
scanner_line_content token=[REDACTED-SENSITIVE-VALUE]
```

### Agent Tool `scan_text`

- Result:

```text
scan_text_raw_secret_present False
scan_text_count 2
scan_text_keys ['detector', 'fingerprint', 'severity', 'span']
scan_text_has_matched_text False
```

### `secret_guard` Hook

- Result:

```text
hook_raw_secret_in_context_repr False
hook_redacted True
hook_content [REDACTED-SENSITIVE-VALUE]
hook_finding_count 2
hook_keys ['detector', 'fingerprint', 'severity', 'span']
hook_has_matched_text False
```

### One-Time Token Flow

- First attempt to issue a token while locked failed with:

```text
Error: Vault is locked; run unlock first
```

- Re-run with explicit `unlock` passed.
- Result:

```text
token_prefix_ok True
token_len 28
second_consume_exit=1
first_consume_raw_dummy_present_expected True
second_consume_raw_dummy_present False
second_consume_rejected True
```

- First consume intentionally returned plaintext dummy data in local terminal output only.
- Second consume rejected the same token:

```text
Error: Token not found or already used
```

### MCP Stdio

- Initial MCP stdio startup and tool listing passed:

```text
server_name vaultknox
server_version 0.7.0
tool_count 6
tool_names vaultknox_status,vaultknox_list,vaultknox_get_metadata,vaultknox_scan,vaultknox_verify,vaultknox_health
```

- Initial MCP status call exposed a blocker:

```text
status_result {"initialized": false, "unlocked": false, "secret_count": 0, "auto_lock_minutes": 15}
```

- Diagnosis: `mcp_server.call_tool()` used `expand_runtime_path()` with no runtime override, so `hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" mcp` started but tool calls still targeted the default vault path.
- Patch applied:
  - `src/vaultknox/mcp_server.py`: added `_mcp_vault_paths()` that honors `VAULTKNOX_RUNTIME_DIR`.
  - `tests/test_mcp_server.py`: added regression test for `VAULTKNOX_RUNTIME_DIR` resolution.

- Focused verification after patch:

```text
17 passed in 0.84s
All checks passed!
```

- MCP stdio re-run after patch:

```text
server_name vaultknox
server_version 0.7.0
tool_count 6
tool_names vaultknox_status,vaultknox_list,vaultknox_get_metadata,vaultknox_scan,vaultknox_verify,vaultknox_health
status_result {"initialized": true, "unlocked": true, "secret_count": 2, "auto_lock_minutes": 15}
status_initialized True
status_secret_count 2
```

## Automated Verification — After MCP Patch

- Command: `/tmp/vk-fork-venv/bin/python -m pytest -q`
- Result: `311 passed in 61.14s`
- Command: `/tmp/vk-fork-venv/bin/python -m ruff check src tests`
- Result: `All checks passed!`
- Command: `bandit -q -r src/vaultknox -f json`
- Result: `bandit_results 40`, `non_low_results 0`
- Command: `pip-audit --path /tmp/vk-fork-venv/lib/python3.11/site-packages`
- Result: `No known vulnerabilities found`; local `vaultknox (0.7.0)` skipped because not on PyPI

## Plan Corrections Made During Execution

- Replaced invalid scanner module-CLI step with source-verified `SecretScanner` API path.
- Confirmed `issue-token` requires an unlocked vault session.
- Confirmed MCP stdio needs `VAULTKNOX_RUNTIME_DIR` to keep local pilots and harness configs out of the default vault path.

## Current Working Tree Changes

- `docs/plans/2026-06-26-vaultknox-local-pilot.md`
- `docs/pilot-evidence/vaultknox-local-pilot-2026-06-26.md`
- `src/vaultknox/mcp_server.py`
- `tests/test_mcp_server.py`

## Verdict

- Automated verification: pass.
- Local dummy-secret runtime pilot through MCP stdio status: pass after MCP runtime-dir patch.
- Raw secret echo in agent-facing scanner/hook/tool surfaces: not observed.
- One-time token single-use behavior: pass.
- Ready for upstream PR from original `harden-agent-secret-safety`: no, because the MCP runtime-dir fix should be included first.
- Recommended next branch action: move the MCP runtime-dir fix onto the security patch branch, rerun full verification, then open upstream PR from the updated security branch. Keep plan/evidence docs off the upstream PR branch unless explicitly desired.
