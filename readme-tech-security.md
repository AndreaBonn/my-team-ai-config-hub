> **Italiano:** [Leggi in italiano](./readme-tech-security-it.md)
>
> **Docs:** [README](./README.md) | [Architecture](./readme-tech-architecture.md)

# Security Audit Notes

## npm audit - Transitive Vulnerabilities (2026-05-17)

`npm audit` reports 12 vulnerabilities (8 low, 4 moderate). All are in transitive dependencies of `next` and `firebase-admin` with no direct exploit path in this application's context.

### Summary

| Package | Severity | Source | Exploitable here? | Reason |
|---------|----------|--------|-------------------|--------|
| postcss (via next) | Moderate | XSS in CSS stringify output | No | We do not process untrusted CSS at runtime; PostCSS runs only at build time |
| @tootallnate/once (via firebase-admin) | Low | Incorrect control flow scoping | No | Transitive dep of http-proxy-agent, not invoked in our code paths |
| Other 10 | Low | Various | No | All are build-time or unused code paths in transitive deps |

### Decision

- These cannot be resolved without major version bumps of `next` or `firebase-admin` that would introduce breaking changes.
- `npm audit fix --force` would downgrade `next` to 9.x which is not acceptable.
- CI runs `npm audit --audit-level=high` to catch any high/critical vulnerability.
- We accept moderate/low transitive risks and monitor for upstream fixes.

### Re-evaluation schedule

Review monthly or when bumping `next` / `firebase-admin` to a new major version.
