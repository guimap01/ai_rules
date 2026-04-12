# SECURITY.md

Supply chain security rules for all AI agents working in this repository.

---

## Before Installing Any Package — MANDATORY

**Never run `pnpm add`, `npm install <pkg>`, or `yarn add` without first auditing the package.**

If you have access to the `mcp__security-audit__audit_nodejs_dependencies` MCP tool, use it:

```
mcp__security-audit__audit_nodejs_dependencies({
  dependencies: { "<package-name>": "<version-or-*>" }
})
```

Check every package in the install command — including indirect additions if known. If the tool reports any HIGH or CRITICAL vulnerabilities, **do not install** and inform the user.

If the MCP tool is unavailable, run `pnpm audit --audit-level=high` after installing and revert (`pnpm remove <pkg>`) if issues are found.

---

## Automated Safeguards Already in Place

| Layer | What it does |
|-------|-------------|
| CI (`deploy.yml`) | `pnpm audit --audit-level=high` blocks all deploys if HIGH/CRITICAL vulns exist |
| Dependabot | Weekly PRs for vulnerable or outdated dependencies across all workspace apps |

Dependabot PRs for security fixes take priority over feature work.
