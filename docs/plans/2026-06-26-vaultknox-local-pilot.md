# VaultKnox Local Pilot Plan

> **For Hermes:** This is a verification plan, not a production install plan. Do not use real credentials. Do not configure live harnesses until the dummy-secret pilot passes.

**Goal:** Prove the patched VaultKnox fork works in real local agent/harness use before claiming it works or opening an upstream PR.

**Architecture:** Run the patched fork from `/home/ubuntu/work/vaultknox-security` inside an isolated temporary VaultKnox home. Exercise the real CLI, autonomous store, MCP stdio server, scanner, secret guard, and dashboard paths using dummy secrets only. Capture evidence from actual command output and transcript-safe tool results.

**Tech Stack:** Python venv, VaultKnox CLI/package from local checkout, pytest, ruff, bandit, pip-audit, MCP stdio, Hermes/Pi/OpenCode harnesses if available.

---

## Assumptions

- Fork branch is `harden-agent-secret-safety`.
- Local checkout is `/home/ubuntu/work/vaultknox-security`.
- Test venv is `/tmp/vk-fork-venv` unless replaced explicitly.
- Pilot vault data lives under `/tmp/vaultknox-pilot-*`, not `~/.vaultknox`.
- Main vault CLI commands use `hermes-vault --runtime-dir "$PILOT_ROOT/runtime"`; `VAULTKNOX_HOME` is only used for the autonomous-store smoke path.
- Only dummy credentials are allowed.
- Any step that prints a raw dummy secret is a failure for that surface unless the command is explicitly a deliberate consume/decrypt test in a local terminal.
- No upstream PR until this plan produces a clean evidence bundle.

## Success Criteria

- Automated checks still pass on the exact branch under test.
- Autonomous store init/set/get works in an isolated temp home.
- Secret scanning returns detector/severity/span/fingerprint/redacted content only, not raw matched text.
- Secret guard blocks or reports dummy secrets without retaining raw secret text in findings.
- One-time token flow works: create, consume once, reject second consume or expired token.
- MCP stdio server starts and exposes expected tools without crashing.
- At least one real harness call path is exercised locally against dummy data.
- Dashboard bootstrap redirects to `/`, stores token as HttpOnly cookie, and escapes hostile labels/filenames.
- Evidence bundle includes exact commands, outputs, branch, commit, and any stop/fail notes.

## Stop Conditions

Stop and patch before continuing if any of these happen:

- Raw dummy secret appears in agent-facing scan output, hook findings, MCP result, logs, or dashboard-rendered HTML where it should be redacted.
- MCP server cannot start from the local checkout.
- Policy or auth behavior allows protected operations without expected identity/token/policy checks.
- Token consume succeeds more than once.
- Dashboard serves bootstrap token in JS-readable page scope.
- Any command attempts to use real `~/.vaultknox`, production configs, or real credentials.
- Static checks introduce new medium/high Bandit findings or dependency vulnerabilities.

---

## Task 1: Reconfirm Exact Branch and Clean Starting State

**Objective:** Verify the pilot runs against the intended fork branch and commit.

**Files:**
- Read only: git metadata in `/home/ubuntu/work/vaultknox-security`

**Steps:**

1. Run:

```bash
cd /home/ubuntu/work/vaultknox-security
git status --short
git branch --show-current
git rev-parse HEAD
git remote -v
```

2. Expected:

```text
working tree clean or only this plan file modified
harden-agent-secret-safety
6ba4c6b65e210601bbd2e02c6aa2f6da98ec6c1b or newer intentional pilot commit
origin points to Gerkinfeltser/Hermes-VaultKnox
upstream points to Ufonik88/Hermes-VaultKnox if configured
```

3. Record output in the evidence bundle.

## Task 2: Run Baseline Automated Verification

**Objective:** Confirm the patched branch still passes automated quality gates before runtime pilot work.

**Files:**
- Read only: repo source/tests

**Steps:**

1. Run:

```bash
cd /home/ubuntu/work/vaultknox-security
/tmp/vk-fork-venv/bin/python -m pytest -q
/tmp/vk-fork-venv/bin/python -m ruff check src tests
/tmp/vk-fork-venv/bin/bandit -q -r src/vaultknox -f json -o /tmp/vk-pilot-bandit.json || true
/tmp/vk-fork-venv/bin/python - <<'PY'
import json
p = '/tmp/vk-pilot-bandit.json'
d = json.load(open(p))
print('bandit_results', len(d.get('results', [])))
print('non_low_results', sum(1 for r in d.get('results', []) if r['issue_severity'] != 'LOW'))
PY
PIPAPI_PYTHON_LOCATION=/tmp/vk-fork-venv/bin/python /tmp/vk-fork-venv/bin/pip-audit --path /tmp/vk-fork-venv/lib/python3.11/site-packages || true
```

2. Expected:

```text
pytest: 310 passed or newer intentional total
ruff: All checks passed
bandit: non_low_results 0
pip-audit: No known vulnerabilities found; local vaultknox package may be skipped if not on PyPI
```

3. Record output in the evidence bundle.

## Task 3: Create an Isolated Pilot Environment

**Objective:** Prevent accidental use of real VaultKnox state.

**Files:**
- Create: `/tmp/vaultknox-pilot-*/`
- Create: `/tmp/vaultknox-pilot-env.sh`

**Steps:**

1. Run:

```bash
PILOT_ROOT=$(mktemp -d /tmp/vaultknox-pilot-XXXXXX)
cat > /tmp/vaultknox-pilot-env.sh <<EOF
export VAULTKNOX_HOME="$PILOT_ROOT/home"
export XDG_CONFIG_HOME="$PILOT_ROOT/config"
export XDG_DATA_HOME="$PILOT_ROOT/data"
export XDG_CACHE_HOME="$PILOT_ROOT/cache"
export VAULTKNOX_RUNTIME_DIR="$PILOT_ROOT/runtime"
export VAULTKNOX_MASTER_PASSWORD="dummy-local-pilot-master-password-not-real"
EOF
. /tmp/vaultknox-pilot-env.sh
mkdir -p "$VAULTKNOX_HOME" "$VAULTKNOX_RUNTIME_DIR" "$XDG_CONFIG_HOME" "$XDG_DATA_HOME" "$XDG_CACHE_HOME"
printf 'PILOT_ROOT=%s\nVAULTKNOX_HOME=%s\nVAULTKNOX_RUNTIME_DIR=%s\n' "$PILOT_ROOT" "$VAULTKNOX_HOME" "$VAULTKNOX_RUNTIME_DIR"
```

2. Expected:

```text
Paths under /tmp/vaultknox-pilot-...
No path under /home/ubuntu/.vaultknox
```

3. Stop if any VaultKnox command writes outside the pilot root.

## Task 4: Exercise Autonomous Store Init/Set/Get

**Objective:** Prove the patched autonomous secrets path works outside pytest.

**Files:**
- Read/write: isolated pilot home only

**Steps:**

1. Run a direct Python smoke test from the local checkout. Adjust class/method names only if source inspection proves the API differs:

```bash
cd /home/ubuntu/work/vaultknox-security
. /tmp/vaultknox-pilot-env.sh
/tmp/vk-fork-venv/bin/python - <<'PY'
import os
import tempfile
from pathlib import Path
from vaultknox.autonomous_secrets import AutonomousSecretsStore

root = Path(os.environ['VAULTKNOX_HOME']) / 'autonomous-smoke'
store = AutonomousSecretsStore(root)
store.initialize(force=True)
store.set('pilot-dummy-id', 'ghp_000000000000000000000000000000000000')
value = store.get('pilot-dummy-id')
print('round_trip_ok', value == 'ghp_000000000000000000000000000000000000')
print('store_root', root)
PY
```

2. Expected:

```text
round_trip_ok True
store_root under /tmp/vaultknox-pilot-...
```

3. If constructor/method names differ, inspect `src/vaultknox/autonomous_secrets.py`, update the command, and record the exact adjustment.

## Task 5: Verify Scanner Redaction in Real CLI/JSON Output

**Objective:** Confirm runtime scanner output does not expose raw dummy secrets.

**Files:**
- Create: `$PILOT_ROOT/fixtures/secret-file.txt`

**Steps:**

1. Create fixture and scan it:

```bash
. /tmp/vaultknox-pilot-env.sh
FIXTURE_DIR=$(dirname "$VAULTKNOX_HOME")/fixtures
mkdir -p "$FIXTURE_DIR"
printf 'token=ghp_111111111111111111111111111111111111\n' > "$FIXTURE_DIR/secret-file.txt"
cd /home/ubuntu/work/vaultknox-security
/tmp/vk-fork-venv/bin/python -m vaultknox.scanner "$FIXTURE_DIR" --json > /tmp/vk-pilot-scan.json
/tmp/vk-fork-venv/bin/python - <<'PY'
import json
raw = open('/tmp/vk-pilot-scan.json').read()
print('raw_secret_present', 'ghp_111111111111111111111111111111111111' in raw)
d = json.loads(raw)
print('finding_count', len(d.get('findings', [])))
for f in d.get('findings', []):
    print('finding_keys', sorted(f.keys()))
    print('line_content', f.get('line_content'))
PY
```

2. Expected:

```text
raw_secret_present False
finding_count >= 1
line_content contains [REDACTED-SENSITIVE-VALUE]
```

3. Stop if raw dummy secret appears in `/tmp/vk-pilot-scan.json`.

## Task 6: Verify Agent Tool scan_text Redaction

**Objective:** Confirm the direct agent-facing scan tool returns fingerprints, not matched text.

**Files:**
- Read only: local checkout

**Steps:**

1. Run:

```bash
cd /home/ubuntu/work/vaultknox-security
/tmp/vk-fork-venv/bin/python - <<'PY'
from vaultknox.hermes_tool import vault_tool
secret = 'ghp_222222222222222222222222222222222222'
result = vault_tool('scan_text', text='token=' + secret)
print('raw_secret_present', secret in repr(result))
print('result', result)
PY
```

2. Expected:

```text
raw_secret_present False
result findings include detector, severity, fingerprint, span
result findings do not include matched_text
```

## Task 7: Verify Secret Guard Hook Findings Redaction

**Objective:** Confirm the hook path does not retain raw secret matches.

**Files:**
- Read only: local checkout

**Steps:**

1. Inspect the hook API if needed:

```bash
cd /home/ubuntu/work/vaultknox-security
/tmp/vk-fork-venv/bin/python - <<'PY'
from pathlib import Path
text = Path('src/vaultknox/hooks/secret_guard.py').read_text()
print(text[:6000])
PY
```

2. Run the hook in its expected shape:

```bash
cd /home/ubuntu/work/vaultknox-security
/tmp/vk-fork-venv/bin/python - <<'PY'
from vaultknox.hooks.secret_guard import handle
secret = 'ghp_333333333333333333333333333333333333'
context = {'content': 'token=' + secret}
handle('message:received', context)
print('raw_secret_in_context', secret in repr(context))
print('context', context)
PY
```

3. Expected:

```text
Findings include fingerprint/span.
Findings do not include matched_text or the raw dummy token.
```

4. Stop if raw dummy secret appears in hook findings.

## Task 8: Verify One-Time Token Flow

**Objective:** Prove tokens are usable once and fail on repeat or expiry.

**Files:**
- Read/write: isolated pilot home only

**Steps:**

1. Discover exact CLI commands without using real state:

```bash
cd /home/ubuntu/work/vaultknox-security
. /tmp/vaultknox-pilot-env.sh
/tmp/vk-fork-venv/bin/hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" --help
/tmp/vk-fork-venv/bin/hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" issue-token --help
/tmp/vk-fork-venv/bin/hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" consume-token --help
/tmp/vk-fork-venv/bin/hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" mcp --help
```

2. Initialize the isolated runtime and add a dummy secret using the CLI. Use stdin only for the dummy master password:

```bash
cd /home/ubuntu/work/vaultknox-security
. /tmp/vaultknox-pilot-env.sh
printf '%s\n%s\n' "$VAULTKNOX_MASTER_PASSWORD" "$VAULTKNOX_MASTER_PASSWORD" \
  | /tmp/vk-fork-venv/bin/hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" init --no-password-check
printf '%s\n' "$VAULTKNOX_MASTER_PASSWORD" \
  | /tmp/vk-fork-venv/bin/hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" add \
      --id pilot-api-key \
      --type api_key \
      --label 'Pilot Dummy API Key' \
      --data '{"service":"pilot","key":"ghp_444444444444444444444444444444444444"}'
```

3. Use discovered/verified commands to:

- Create one-time token for it.
- Consume token once and verify expected dummy plaintext appears only in the local terminal command meant to reveal it.
- Consume same token again and verify rejection.
- If TTL option exists, create short TTL token and verify expiry rejection.

4. Expected:

```text
first consume: success
second consume: rejected
expired consume: rejected if TTL supported
logs/audit output: no raw dummy secret
```

5. Record exact commands because CLI naming may differ from expectation.

## Task 9: Start MCP Stdio Server from Local Checkout

**Objective:** Prove the patched fork can run as an MCP stdio server without crashing.

**Files:**
- Read/write: isolated pilot home only

**Steps:**

1. Discover MCP entry point:

```bash
cd /home/ubuntu/work/vaultknox-security
. /tmp/vaultknox-pilot-env.sh
/tmp/vk-fork-venv/bin/python - <<'PY'
import importlib.metadata as md
for ep in md.entry_points().select(group='console_scripts'):
    if 'vault' in ep.name.lower() or 'knox' in ep.name.lower() or 'hermes' in ep.name.lower():
        print(ep.name, '->', ep.value)
PY
/tmp/vk-fork-venv/bin/hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" mcp --help
```

2. Start the MCP server with a minimal stdio initialize request if documented by the project. Prefer a purpose-built MCP inspector/client if already installed; otherwise use a short Python subprocess smoke test that sends `initialize` and waits for a JSON-RPC response.

3. Expected:

```text
server starts
initialize response received
expected tool list can be queried
no traceback
```

4. Stop if the MCP server reads or writes outside pilot root.

## Task 10: Exercise One Real Harness Path

**Objective:** Prove this is not just repo-internal testing; one actual agent harness can call the MCP server or equivalent tool path.

**Files:**
- Create: temporary harness config under `/tmp/vaultknox-pilot-*` only, unless explicitly approved otherwise

**Preferred order:**

1. Hermana local MCP config in temp/scratch mode if Hermes supports per-run MCP config without modifying live profile.
2. OpenCode temp config pointing at local MCP stdio command.
3. Pi temp config pointing at local MCP stdio command.

**Required behavior:**

- Harness discovers VaultKnox tools.
- Harness reads a masked secret or creates/consumes a dummy one-time token according to policy.
- Harness transcript does not include raw dummy secret except for an explicitly approved local reveal command.

**Stop condition:** Do not edit live `~/.hermes/config.yaml`, Pi config, or OpenCode config during this task without an explicit follow-up approval.

## Task 11: Dashboard Hostile Data Smoke Test

**Objective:** Prove dashboard HTML/JS no longer exposes bootstrap token and escapes hostile API-fed strings.

**Files:**
- Read/write: isolated pilot home only

**Steps:**

1. Start dashboard bound to localhost only, using pilot env and `hermes-vault --runtime-dir "$VAULTKNOX_RUNTIME_DIR" dashboard --host 127.0.0.1 --port 0 --no-open`.
2. Request `/?token=<dummy-token>` with curl.
3. Verify response:

```text
HTTP 302 to /
Set-Cookie includes HttpOnly
body does not include bootstrap token
```

4. Create hostile dummy label/file values such as:

```text
<img src=x onerror=alert(1)>
```

5. Load affected API/dashboard paths and verify rendered HTML/JS escapes the string or client-side `esc()` handles it before insertion.

6. Stop if hostile string is inserted unescaped into `innerHTML`.

## Task 12: Write Evidence Bundle

**Objective:** Produce the artifact that supports the final claim.

**Files:**
- Create: `docs/pilot-evidence/vaultknox-local-pilot-YYYY-MM-DD.md`

**Evidence format:**

```markdown
# VaultKnox Local Pilot Evidence — YYYY-MM-DD

## Scope
- Repo:
- Branch:
- Commit:
- Pilot root:
- Real credentials used: no

## Automated Verification
- pytest:
- ruff:
- bandit:
- pip-audit:

## Runtime Checks
- autonomous store:
- scanner redaction:
- scan_text redaction:
- secret_guard redaction:
- one-time token flow:
- MCP stdio:
- harness path:
- dashboard:

## Failures / Deviations
- none, or exact blocker

## Verdict
- Ready for upstream PR: yes/no
- Ready for private fork pilot with real credentials: yes/no
- Required follow-up patches:
```

## Task 13: Decide Next Gate

**Objective:** Make the next public/private claim match the evidence.

**If all tasks pass:**

- Claim: “Patched fork passes automated checks and local dummy-secret MCP/harness pilot.”
- Next action: open upstream PR with concise security-focused body and evidence link.

**If any task fails:**

- Claim: “Automated checks pass, but local pilot found blocker(s).”
- Next action: patch branch again, add regression tests, rerun this plan from the failed task onward.

**If harness config requires live profile edits:**

- Claim: “Repo-local pilot passed up to MCP stdio; live harness config requires operator approval.”
- Next action: request approval for exactly one harness config change.

---

## Final Operator Rule

Until this plan passes, the correct status is:

```text
Patched fork passes automated verification and is ready for local dummy-secret pilot.
```

After this plan passes, the correct status becomes:

```text
Patched fork passes automated verification and local dummy-secret pilot.
```
