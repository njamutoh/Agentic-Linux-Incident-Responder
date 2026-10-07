# Pretzel API Lab Service Remediation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Recover the intentionally simulated `pretzel-api.service` failure by creating the missing lab executable at the path already configured in the unit.

**Architecture:** Keep the lab fully isolated from the real Pretzel Shop application under `/home/devopsy/pretzel-shop`. Do not repoint the unit to any existing application. Preserve the existing `ExecStart=/opt/pretzel/bin/pretzel-api` contract and satisfy it with a minimal, root-owned, non-destructive executable that stays alive so systemd can maintain `active/running` state.

**Tech Stack:** RHEL 9.8, systemd, POSIX shell/bash, coreutils, journalctl, systemctl.

**Spec:** User incident instructions in this session; evidence preserved in `evidence/before.txt`.

## Global Constraints

- Start with read-only investigation; do not change system state before approval.
- Preserve pre-remediation evidence in `evidence/before.txt`.
- Submit this remediation plan for human review and wait for approval before changes.
- Do not use the real Pretzel Shop backend at `/home/devopsy/pretzel-shop` as the remediation target.
- Do not modify, start, stop, or depend on the real Pretzel Shop application.
- Do not change `pretzel-api.service` to point at an unrelated application.
- Do not disable SELinux.
- Do not disable the firewall.
- Do not install unnecessary software.
- Do not make unrelated changes.
- Do not perform destructive actions.
- Implement only approved actions after approval.
- Back up the original systemd unit before remediation.
- Verify service state at least three consecutive times after remediation.
- Verify the post-remediation journal after restart.
- Preserve post-remediation evidence in `evidence/after.txt`.
- Create `reports/incident-report.md` after verification.

## Review Focus

- Missing executable: `/opt/pretzel/bin/pretzel-api` must exist and be executable before restart.
- Lab isolation: remediation must not reference `/home/devopsy/pretzel-shop` or any real Pretzel Shop backend process.
- Minimal behavior: the lab executable should only log startup/shutdown and wait; it must not bind ports, modify data, call networks, or start dependencies.
- Actual recovery: verify `ActiveState=active` and `SubState=running` three consecutive times, not merely that `systemctl restart` exits successfully.
- Journal verification: confirm no fresh `203/EXEC` or `Failed to locate executable` messages after the remediation restart.

---

## Evidence Summary

Observed before remediation:

- `systemctl status pretzel-api.service` reports `status=203/EXEC` and `Result: exit-code`.
- `journalctl -u pretzel-api.service` repeatedly reports `Failed to locate executable /opt/pretzel/bin/pretzel-api: No such file or directory` and `Failed at step EXEC spawning /opt/pretzel/bin/pretzel-api`.
- `systemctl cat pretzel-api.service` shows `ExecStart=/opt/pretzel/bin/pretzel-api`.
- `/opt/pretzel`, `/opt/pretzel/bin`, and `/opt/pretzel/bin/pretzel-api` do not exist.
- `/etc/systemd/system/pretzel-api.service` exists and is enabled through `multi-user.target.wants`.

Root cause: the intentionally created lab unit points `ExecStart` at `/opt/pretzel/bin/pretzel-api`, but that executable path does not exist. The failure occurs before any application code could start because systemd cannot exec the configured command.

Out-of-scope finding: `/home/devopsy/pretzel-shop` contains a separate real Pretzel Shop application. It must not be used, modified, started, stopped, or referenced as the replacement executable for this lab incident.

## File/System Changes to Make After Approval

- Back up: `/etc/systemd/system/pretzel-api.service` to `/etc/systemd/system/pretzel-api.service.incident-20261006.bak`.
- Create directory: `/opt/pretzel/bin`.
- Create executable: `/opt/pretzel/bin/pretzel-api`.
- Do not modify `/etc/systemd/system/pretzel-api.service` unless verification proves the on-disk unit differs from `evidence/before.txt`; if it differs, stop and return to planning.
- Runtime commands after executable creation:
  - `systemctl restart pretzel-api.service`
- Create/update local documentation artifacts:
  - `evidence/after.txt`
  - `reports/incident-report.md`

## Task 1: Create the missing isolated lab executable

**Files:**
- Read/backup: `/etc/systemd/system/pretzel-api.service`
- Create: `/opt/pretzel/bin/pretzel-api`

**Interfaces:**
- Consumes: existing unit contract `ExecStart=/opt/pretzel/bin/pretzel-api`.
- Produces: a safe executable at that exact path that remains running until systemd stops it.

- [ ] **Step 1: Confirm the unit still matches the established failure contract**

```bash
systemctl cat pretzel-api.service
systemctl show pretzel-api.service -p FragmentPath -p ExecStart --no-pager
```

Expected: the unit still contains `ExecStart=/opt/pretzel/bin/pretzel-api`. If the unit differs from `evidence/before.txt`, stop and revise the plan before making changes.

- [ ] **Step 2: Back up the original systemd unit before remediation**

```bash
sudo cp -a /etc/systemd/system/pretzel-api.service /etc/systemd/system/pretzel-api.service.incident-20261006.bak
sudo cmp -s /etc/systemd/system/pretzel-api.service /etc/systemd/system/pretzel-api.service.incident-20261006.bak
sudo stat /etc/systemd/system/pretzel-api.service.incident-20261006.bak
```

Expected: backup exists and `cmp` exits successfully.

- [ ] **Step 3: Create the lab executable directory**

```bash
sudo install -d -o root -g root -m 0755 /opt/pretzel/bin
```

Expected: `/opt/pretzel/bin` exists, owned by `root:root`, mode `0755`.

- [ ] **Step 4: Create the minimal non-destructive lab executable**

```bash
sudo tee /opt/pretzel/bin/pretzel-api >/dev/null <<'SCRIPT'
#!/usr/bin/env bash
set -euo pipefail

echo "pretzel-api lab service started: pid=$$"

trap 'echo "pretzel-api lab service stopping"; exit 0' TERM INT

while true; do
  sleep 3600 &
  wait "$!"
done
SCRIPT
sudo chown root:root /opt/pretzel/bin/pretzel-api
sudo chmod 0755 /opt/pretzel/bin/pretzel-api
```

Expected: executable is a simple wait loop with signal handling. It does not bind ports, modify files, access `/home/devopsy/pretzel-shop`, start dependencies, or perform network calls.

- [ ] **Step 5: Verify the executable path and syntax before restart**

```bash
ls -l /opt/pretzel/bin/pretzel-api
bash -n /opt/pretzel/bin/pretzel-api
```

Expected: file is executable and `bash -n` exits successfully.

## Task 2: Restart only the lab service and verify recovery

**Files:**
- Create/update: `evidence/after.txt`

**Interfaces:**
- Consumes: `/opt/pretzel/bin/pretzel-api` from Task 1.
- Produces: objective post-remediation evidence.

- [ ] **Step 1: Restart only `pretzel-api.service`**

```bash
sudo systemctl restart pretzel-api.service
```

Expected: command exits successfully. Do not restart unrelated services.

- [ ] **Step 2: Verify actual systemd state three consecutive times**

```bash
for i in 1 2 3; do
  echo "--- service state attempt $i ---"
  systemctl is-active pretzel-api.service
  systemctl show pretzel-api.service -p ActiveState -p SubState -p Result -p ExecMainStatus -p ExecStart -p MainPID --no-pager
  sleep 2
done
systemctl status pretzel-api.service --no-pager --full
```

Expected each attempt: `is-active` prints `active`; `ActiveState=active`; `SubState=running`; `ExecStart` still points to `/opt/pretzel/bin/pretzel-api`; no new `203/EXEC`.

- [ ] **Step 3: Verify post-remediation journal**

```bash
journalctl -u pretzel-api.service --no-pager -n 120 -o short-iso
```

Expected: after the remediation restart timestamp, journal shows the lab service startup message and no fresh `Failed to locate executable /opt/pretzel/bin/pretzel-api`, `Failed at step EXEC`, or `status=203/EXEC` messages.

- [ ] **Step 4: Save post-remediation evidence**

```bash
mkdir -p evidence
{
  echo '# Post-remediation evidence: pretzel-api.service lab incident'
  date -Is
  echo '## executable path'
  ls -l /opt/pretzel/bin/pretzel-api || true
  stat /opt/pretzel/bin/pretzel-api || true
  echo '## unit'
  systemctl cat pretzel-api.service || true
  echo '## service state attempts'
  for i in 1 2 3; do
    echo "--- service state attempt $i ---"
    systemctl is-active pretzel-api.service || true
    systemctl show pretzel-api.service -p ActiveState -p SubState -p Result -p ExecMainStatus -p ExecStart -p MainPID --no-pager || true
    sleep 2
  done
  echo '## status'
  systemctl status pretzel-api.service --no-pager --full || true
  echo '## recent journal'
  journalctl -u pretzel-api.service --no-pager -n 120 -o short-iso || true
} > evidence/after.txt
```

## Task 3: Roll back if verification fails or the lab needs reset

**Files:**
- Remove: `/opt/pretzel/bin/pretzel-api`
- Remove if empty: `/opt/pretzel/bin`, `/opt/pretzel`
- Restore: `/etc/systemd/system/pretzel-api.service` from backup

**Interfaces:**
- Consumes: backup unit at `/etc/systemd/system/pretzel-api.service.incident-20261006.bak`.
- Produces: pre-remediation lab state with the missing executable failure restored.

- [ ] **Step 1: Restore the original unit from backup**

```bash
sudo cp -a /etc/systemd/system/pretzel-api.service.incident-20261006.bak /etc/systemd/system/pretzel-api.service
sudo systemctl daemon-reload
```

Expected: original unit content is restored. This is included even though the planned remediation does not modify the unit.

- [ ] **Step 2: Remove the lab executable created by remediation**

```bash
sudo rm -f /opt/pretzel/bin/pretzel-api
sudo rmdir /opt/pretzel/bin /opt/pretzel 2>/dev/null || true
```

Expected: `/opt/pretzel/bin/pretzel-api` no longer exists. Empty directories created solely for the lab executable are removed; non-empty directories are left intact.

- [ ] **Step 3: Restart only the lab service to return to the original configured failure state**

```bash
sudo systemctl restart pretzel-api.service || true
```

Expected: the service may return to the original `203/EXEC` failure mode. Rollback success means the original unit and missing-executable state have been restored, not that the incident is fixed.

- [ ] **Step 4: Capture rollback evidence**

```bash
{
  echo '# Rollback evidence: pretzel-api.service lab incident'
  date -Is
  echo '## restored unit'
  systemctl cat pretzel-api.service || true
  echo '## executable path'
  ls -l /opt/pretzel/bin/pretzel-api || true
  stat /opt/pretzel/bin/pretzel-api || true
  echo '## service state'
  systemctl status pretzel-api.service --no-pager --full || true
  systemctl show pretzel-api.service -p ActiveState -p SubState -p Result -p ExecMainStatus -p ExecStart -p MainPID --no-pager || true
  echo '## recent journal'
  journalctl -u pretzel-api.service --no-pager -n 120 -o short-iso || true
} > evidence/rollback.txt
```

Then return to systematic debugging with the new journal evidence. Do not attempt unrelated fixes.

## Task 4: Document incident

**Files:**
- Create/update: `reports/incident-report.md`

**Interfaces:**
- Consumes: `evidence/before.txt`, `evidence/after.txt`, investigation notes, and approved remediation actions.
- Produces: final incident report with required sections.

- [ ] **Step 1: Create report with required sections**

Include:

```markdown
# Incident Report: pretzel-api.service Lab Failure

## Incident summary

## Symptoms

## Investigation performed

## Evidence

## Root cause

## Remediation

## Verification

## Rollback procedure

## Scope note

## Preventive recommendation
```

- [ ] **Step 2: Populate report from evidence files**

Use exact observed facts from `evidence/before.txt` and `evidence/after.txt`. State explicitly that the real Pretzel Shop application under `/home/devopsy/pretzel-shop` was not used or changed. Do not claim recovery unless Task 2 confirms active service state three consecutive times and post-remediation journal review shows no fresh `203/EXEC`.

## Self-Review

- Spec coverage: plan preserves evidence before/after, asks for approval before changes, avoids the real Pretzel Shop backend, avoids SELinux/firewall/package changes, backs up the unit, creates only the missing lab executable, verifies service state three times, checks post-remediation journal, provides exact rollback, and documents the incident.
- Placeholder scan: no TBD/TODO placeholders remain.
- Command consistency: service name, executable path, backup path, and rollback commands are consistent across tasks.
- Review focus: the highest-risk failure modes are addressed by executable creation, isolation constraints, verification steps, or rollback steps.
