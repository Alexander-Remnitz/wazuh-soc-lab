# Privileged Command Detection — Wazuh Rule 5402

Live verification completed: 2026-10-02.

- Agent: `mr-axe` (`10.66.66.20`)
- Rule: `5402` — Successful sudo to ROOT executed
- MITRE ATT&CK: `T1548.003` — Sudo and Sudo Caching
- Source user: `labadmin`
- Destination user: `root`
- Command: `/usr/bin/id`
- Log source: `journald`
- Verified event time: `2026-10-02 07:49:56 UTC`

The live Wazuh alert contained the full executed command and confirmed successful sudo-to-root activity on the target.
