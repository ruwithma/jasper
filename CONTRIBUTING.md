# Contributing to jasper

Thanks for helping improve jasper. It's a single Bash script — no build step,
no dependencies beyond the standard Kali toolset.

## Ground rules

- **Authorized use only.** jasper is for CTFs, HackTheBox, and engagements you
  have written permission to test. Contributions that only make sense for
  unauthorized targeting won't be merged.
- Keep it to the **standard Kali toolset**. New tool dependencies should degrade
  gracefully — if the tool is missing, the module skips (add it to the preflight
  `TOOL_AREA` map so users see why a module was blind).
- One script. Don't split it up.

## Before opening a PR

1. `bash -n jasper` must pass (syntax check).
2. Run `shellcheck jasper` if you have it; keep new warnings at zero.
3. Test against a box (HTB retired / local VM) and paste the relevant output.
4. Match the existing style: `log`/`ok`/`warn`/`err` helpers, `run_bg` for
   parallel jobs, per-service module writes `services/<port>_<name>.txt`.

## Adding a service module

1. Write `enum_<svc>() { local p=$1; ... > "$SVCDIR/${p}_<svc>.txt"; }`.
2. Wire it into the dispatch `case` in PHASE 5 (service name first, port fallback).
3. If it produces high-signal findings, add a pattern to `prioritize()` so it
   surfaces in the **START HERE** list.

## Reporting bugs

Open an issue with: target OS/service, the jasper command you ran, the relevant
`SUMMARY.txt` / module output (redact real IPs/creds), and what you expected.
