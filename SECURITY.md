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

## pnpm Overrides — Check Before Adding Direct Deps

The root `package.json` has a `pnpm.overrides` section that pins versions for certain packages across the entire monorepo (e.g. `multer: ">=2.1.1"`, `axios: ">=1.15.0"`). When adding a package as a **direct dependency** of any app, its version specifier must match the override specifier exactly — otherwise CI fails with:

```
ERR_PNPM_OUTDATED_LOCKFILE  specifiers in the lockfile don't match specifiers in package.json
```

**Steps when adding a new dep:**

1. Check root `package.json` → `pnpm.overrides` for the package name.
2. If an override exists, use that exact specifier (e.g. `>=2.1.1`) in the app's `package.json`, not `^x.y.z`.
3. Run `pnpm install --frozen-lockfile` locally to verify before committing.

---

## Automated Safeguards Already in Place

| Layer | What it does |
|-------|-------------|
| CI (`deploy.yml`) | `pnpm audit --audit-level=high` blocks all deploys if HIGH/CRITICAL vulns exist |
| Dependabot | Weekly PRs for vulnerable or outdated dependencies across all workspace apps |

Dependabot PRs for security fixes take priority over feature work.
