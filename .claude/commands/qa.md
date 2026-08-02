---
description: Run the full local quality gate — build, format, test, dependency audit, secret scan, and SonarQube. Never pushes.
---

# /qa

The complete local gate. Run it before [`/review`](review.md) and before proposing any push.
**This command never pushes.**

## Build, format, test

```bash
dotnet restore
dotnet build --no-incremental                      # warnings are errors
dotnet format --verify-no-changes                  # style gate
dotnet test --collect:"XPlat Code Coverage"        # unit + integration + architecture
dotnet list package --vulnerable --include-transitive
```

## Secrets

```bash
gitleaks detect --no-banner --redact               # or: .claude/hooks/secret-scan.sh
```

## Static analysis

Run the SonarQube scanner and read the result — see
[`../standards/sonarqube.md`](../standards/sonarqube.md) for the exact invocation and the MCP
read path.

## Enforce

- **Any open Blocker / Critical / Major → fix, rerun, rescan.** Repeat until zero remain.
- Coverage on new code ≥80%.
- Any high-severity dependency advisory → resolve or document an accepted risk.
- Any secret finding → **stop**. Rotate the secret first, then remove it from history.
- Minor / Info → triage and record.

## Report

Print each step's real result — not a summary of what you expected. State the SonarQube gate
status, the open Blocker/Critical/Major count, coverage on new code, and the remaining risks.

**Then stop and ask for explicit push approval.** Do not push.
