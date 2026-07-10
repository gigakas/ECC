---
name: config-health
description: Runtime configuration integrity monitoring for Claude Code. Detects config drift, rule staleness, and hook wiring gaps during sessions. Complements delivery-gate (session-end hard block) with session-duration soft monitoring.
metadata:
  origin: ECC
---

# Config Health — Runtime Configuration Integrity Monitor

A real-time monitoring layer for Claude Code configuration integrity. While delivery-gate blocks bad output at session end, config-health catches config problems during the session — when they're still cheap to fix.

## Why

delivery-gate (Stop hook) checks whether work is complete. But by the time the Stop hook fires, subtle config problems have already affected the entire session. Config-health monitors during the session — when problems are cheap to fix.

Together: config-health = process layer (soft monitoring). delivery-gate = output layer (hard blocking). Defense in depth.

## What It Monitors

1. Config Rule Health (SessionStart): scans rules, checks hook wiring, detects dead rules
2. Hook Wiring Audit (PreToolUse): verifies scripts exist and are wired before high-risk calls
3. Guard Script Staleness: checks maintenance windows of guard scripts

## Install

```json
{
  "hooks": {
    "SessionStart": [{
      "hooks": [{"type": "command", "command": "python3 ~/.claude/scripts/config-health.py --startup", "timeout": 5000}]
    }],
    "PreToolUse": [{
      "matcher": "Edit|Write",
      "hooks": [{"type": "command", "command": "python3 ~/.claude/scripts/config-health.py --pretool", "timeout": 3000}]
    }]
  }
}
```

## Integration With delivery-gate

Session Start → config-health scans rules/hooks (soft warnings, never blocks)
Session Work → config-health monitors wiring (PreToolUse guard)
Session End → delivery-gate verifies capture/disk (hard block, exit 2)

Boundary: "Can this be fixed retroactively?" Config drift = yes → warn. Missing growth-log = no → block.

## Design Principles

1. Never block on process monitoring
2. Check what delivery-gate can't
3. Zero false positives on hook wiring
4. Organic growth — each channel added when a failure mode was observed

## Files

| File | Purpose | Hook |
|------|---------|------|
| scripts/config-health.py | Rule health, hook audit, staleness | SessionStart + PreToolUse |
