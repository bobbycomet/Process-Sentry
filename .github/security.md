# Security Policy

## Supported Versions

ProcessSentry v3 is currently the only actively maintained version.

| Version | Supported          |
|---------|--------------------|
| 3.x     | :white_check_mark: |
| < 3.0   | :x:                |

## Reporting a Vulnerability

**ProcessSentry** is a small, solo-developed open-source project. I take security seriously, even though I cannot offer bounties or a full-time security team.

If you discover a security vulnerability, please report it responsibly so I can address it quickly and privately.

### Preferred way to report
- Join my Discord server: [https://discord.gg/fMCpeNCxhv](https://discord.gg/fMCpeNCxhv)
- Send a direct message to **Bobby Comet** (or ping me in the appropriate channel) with a clear title like **"[Security] Vulnerability in ProcessSentry"**.

Please include as much of the following information as possible:
- Description of the vulnerability and its potential impact
- Steps to reproduce the issue
- Affected versions of ProcessSentry
- Any suggested fixes or mitigations (optional but very helpful)
- Whether the issue has already been disclosed publicly

I aim to acknowledge your report within **48 hours**, and will work with you on a fix and coordinated disclosure.

### What to expect
- I will confirm receipt of the report.
- I will investigate and keep you updated on progress.
- Once fixed, I will release a patch and credit you (unless you prefer to stay anonymous).
- I generally aim to release security fixes within **7–14 days** of confirmation, depending on severity.

**Do not** open a public GitHub issue or discuss the vulnerability publicly until a fix has been released and you've coordinated with me.

## Security Considerations for ProcessSentry

ProcessSentry runs as a **root** systemd service because it needs privileges to:
- Adjust process priorities (`nice` / `ionice`)
- Manage cgroups (CPU quotas and I/O weights)
- Read process information for all users via `/proc`

### Important notes
- It **never kills processes** — it only lowers priority or applies cgroup limits.
- It has built-in safeguards to avoid touching critical system processes (desktop shell, audio servers, compositors, games when marked).
- The daemon is written in Python and uses minimal dependencies (`psutil` and `PyYAML`).
- Polling intervals are configurable and kept lightweight by design to minimize overhead.

### Recommendations for users
- Review the default exclusions in `/etc/process-sentry/config.yaml` and adjust them if needed.
- Use the kill-switch (`/etc/no-auto-throttle`) during sensitive benchmarks or testing.
- Keep your system and kernel updated.
- Consider running with additional hardening (AppArmor/SELinux profiles, systemd sandboxing) if you have a high-security environment.

## Disclosure Policy

I follow **responsible coordinated disclosure**:
- I will not publicly disclose a vulnerability until a fix is available (or a reasonable timeline has passed).
- You are welcome to disclose publicly after the fix is released and you've given me a reasonable time to patch.

## Security Updates

Security-related releases will be clearly marked in the GitHub Releases and changelog. I recommend keeping ProcessSentry updated, especially if you use it on a gaming or daily-driver system.

## Thank You

Thank you for helping keep ProcessSentry (and the broader Linux desktop community) more secure. Every responsible report is greatly appreciated.

---
