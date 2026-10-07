# Incident Report: pretzel-api.service Lab Failure

## Incident summary

`pretzel-api.service` failed in an intentional lab scenario because its configured executable path did not exist. The final approved remediation was executed by the agent with the least-privilege helper:

```bash
sudo -n /usr/local/sbin/pretzel-lab-remediate
```

The helper recreated the missing lab executable at `/opt/pretzel/bin/pretzel-api` and returned the service to `active/running` state.

## Symptoms

Pre-remediation evidence in `evidence/before.txt` showed:

- `pretzel-api.service` was failing with `status=203/EXEC`.
- `systemctl show` reported `Result=exit-code` and `ExecMainStatus=203`.
- The unit had `ExecStart=/opt/pretzel/bin/pretzel-api`.
- The journal reported `Failed to locate executable /opt/pretzel/bin/pretzel-api: No such file or directory` and `Failed at step EXEC spawning /opt/pretzel/bin/pretzel-api`.
- `/opt/pretzel`, `/opt/pretzel/bin`, and `/opt/pretzel/bin/pretzel-api` did not exist in the original captured evidence.

The lab incident was intentionally recreated before the final agent-executed remediation. Immediately before the final helper run on `2026-10-06T20:37:42-06:00`, the service was again `activating`, `ExecMainStatus=203`, and `/opt/pretzel/bin/pretzel-api` was missing.

## Investigation performed

The investigation reviewed:

- `evidence/before.txt`
- current `systemctl show` output for `pretzel-api.service`
- current filesystem state for `/opt/pretzel/bin/pretzel-api`
- recent `journalctl -u pretzel-api.service` output
- post-remediation service state and journal output

A separate real Pretzel Shop application exists under `/home/devopsy/pretzel-shop`, but it is out of scope for this intentionally created lab service.

## Evidence

Pre-remediation evidence:

- `evidence/before.txt`
- Unit path: `/etc/systemd/system/pretzel-api.service`
- Configured command: `ExecStart=/opt/pretzel/bin/pretzel-api`
- Failure: `status=203/EXEC`
- Missing path: `/opt/pretzel/bin/pretzel-api`

Final agent-executed remediation evidence:

- Command executed: `sudo -n /usr/local/sbin/pretzel-lab-remediate`
- Helper exit code: `0`
- Post-remediation evidence file: `evidence/after.txt`
- `/opt/pretzel/bin/pretzel-api` exists as `root:root`, mode `0755`.
- `bash -n /opt/pretzel/bin/pretzel-api` succeeded during evidence collection.
- The unit still points to `/opt/pretzel/bin/pretzel-api`.
- `systemctl is-active pretzel-api.service` returned `active` three consecutive times.
- `systemctl show` returned `ActiveState=active`, `SubState=running`, `Result=success`, `ExecMainStatus=0`, and `MainPID=299383` during verification.
- `systemctl status` showed `Active: active (running)` with `bash /opt/pretzel/bin/pretzel-api` and child `sleep 3600`.
- Journal output after the final remediation showed `pretzel-api lab service started: pid=299383`.

## Root cause

`pretzel-api.service` was configured with:

```ini
ExecStart=/opt/pretzel/bin/pretzel-api
```

That executable did not exist. systemd therefore failed before any process could start, producing `203/EXEC` and repeated restart attempts.

## Remediation

The final approved remediation was executed by the agent using the least-privilege helper:

```bash
sudo -n /usr/local/sbin/pretzel-lab-remediate
```

The helper completed successfully with exit code `0`. It restored the lab service by creating the missing executable at `/opt/pretzel/bin/pretzel-api` and restarting/recovering the lab service according to the approved remediation flow.

The remediation did not modify the systemd unit and did not point the service at any existing application.

## Verification

Verification in `evidence/after.txt` confirmed:

- `/opt/pretzel/bin/pretzel-api` exists and is executable.
- The systemd unit remains unchanged and still uses `ExecStart=/opt/pretzel/bin/pretzel-api`.
- Three consecutive service-state checks returned:
  - `systemctl is-active pretzel-api.service`: `active`
  - `ActiveState=active`
  - `SubState=running`
  - `Result=success`
  - `ExecMainStatus=0`
- `systemctl status pretzel-api.service` showed `Active: active (running)`.
- The process tree showed `bash /opt/pretzel/bin/pretzel-api` and a child `sleep 3600` process.
- Journal output after the final remediation showed the lab startup message. Earlier `203/EXEC` entries remain in the recent journal as historical pre-remediation evidence, but no new exec failure appears after the helper-created startup at `2026-10-06 20:37:43`.

## Rollback procedure

To return the lab to the original missing-executable failure state:

```bash
sudo cp -a /etc/systemd/system/pretzel-api.service.incident-20261006.bak /etc/systemd/system/pretzel-api.service
sudo systemctl daemon-reload
sudo rm -f /opt/pretzel/bin/pretzel-api
sudo rmdir /opt/pretzel/bin /opt/pretzel 2>/dev/null || true
sudo systemctl restart pretzel-api.service || true
```

Rollback success means the original unit and missing-executable state have been restored. The service may then return to the original `203/EXEC` failure mode.

## Scope note

The real Pretzel Shop application under `/home/devopsy/pretzel-shop` was not used, modified, started, stopped, or referenced as the replacement executable. This remediation remained isolated to the intentionally simulated lab service failure.

## Preventive recommendation

For lab services, provision the expected executable path before enabling or starting the unit. A pre-flight check such as `test -x /opt/pretzel/bin/pretzel-api` would have detected this failure before the service entered a restart loop.
