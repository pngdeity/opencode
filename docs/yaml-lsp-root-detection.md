# YAML LSP Root Detection — Background Research

## Summary

OpenCode's built-in `yaml-ls` LSP server only activates for `.yml`/`.yaml` files when a Node.js package manager lock file (`package-lock.json`, `bun.lockb`, `bun.lock`, `pnpm-lock.yaml`, or `yarn.lock`) exists in the directory hierarchy. This makes YAML LSP functionally dead for any non-Node.js project. The same restrictive root detection applies to `typescript` LSP, though `package.json` is a reasonable proxy for TypeScript projects.

## Root Cause

**File:** `packages/opencode/src/lsp/server.ts`
**Line:** ~1399 (commit `b46cec2a7` on `dev` branch)

```ts
export const YamlLS: Info = {
    id: "yaml-ls",
    extensions: [".yaml", ".yml"],
    root: NearestRoot(["package-lock.json", "bun.lockb", "bun.lock",
                        "pnpm-lock.yaml", "yarn.lock"]),
    async spawn(root) {
        // ...
    },
}
```

The `NearestRoot` function walks UP the directory tree from the opened file, looking for any of the specified marker files. If none are found, `root` returns `undefined` and the LSP never spawns. There is no fallback.

## Why This Is Wrong

YAML files are universal configuration files. They exist in:
- Kubernetes manifests (no lock file)
- GitHub Actions workflows (no lock file)
- Docker Compose files (no lock file)
- Ansible playbooks (no lock file)
- CI/CD pipeline configs (no lock file)
- APM `apm.yml` manifests (no lock file)
- OpenAPI specifications (no lock file)

None of these project types have `package-lock.json` or similar. The current root detection is a Node.js-centric assumption that breaks YAML LSP for the majority of real-world YAML use cases.

## Comparison: Other LSP root detections

| LSP | Root detection | Sensible? |
|-----|---------------|-----------|
| `bash` | `ctx.directory` (editing directory) | Yes — bash scripts are standalone |
| `gopls` | `go.mod` | Yes — Go projects always have go.mod |
| `pyright` | `pyproject.toml`, `setup.py`, etc. | Yes — Python markers |
| `terraform` | `.terraform`, `.terragrunt` | Yes — Terraform markers |
| `lua-ls` | `.luarc.json`, `.luacheckrc`, etc. | Yes — Lua markers |
| `yaml-ls` | `package-lock.json`, `bun.lockb`, etc. | **No — YAML is not Node.js-specific** |
| `typescript` | Same as yaml-ls | Mostly fine — `package.json` is standard for TS |

## Prior Art in the Issue Tracker

### Issue #7842: "YAML LSP not working"
- Filed: Jan 11, 2026
- Author: gitaupi
- Status: **Auto-closed** after 90 days of inactivity
- Closure reason: Bot comment from @thdxr — "To stay organized, issues are automatically closed after 90 days of no activity."
- The issue body documented exact symptoms matching this root cause.

### PR #6986: "fix: yaml-ls initialization and bootstrap environment issues"
- Submitted: Jan 5, 2026 by @processtrader
- Status: **Never merged**, auto-closed after 60 days
- Changes: 25 lines, 1 file (`server.ts`)
- Root detection fix: Added `.yamllint`, `kustomization.yaml`, `docker-compose.yaml` to root markers
- Why it died: The PR body used `Issue: https://github.com/...` URL format instead of `Fixes #7842` keyword. The issue-link bot applied `needs:issue` label. Zero maintainers ever reviewed the code.
- Screenshots and test evidence were included but wasted on a regex mismatch.

### Issue #11509: "Allow to pass settings to the yaml-ls"
- Filed: Jan 31, 2026 by nevmerzhitsky
- Status: **Auto-closed** from inactivity
- Related but different concern (YAML LSP schema settings).

### Issue #18694: TypeScript LSP fails in monorepos
- Status: **Still open** (as of research date)
- Same root cause: `NearestRoot(["package-lock.json", ...])` finds nothing in monorepo subdirectories.

## Technical Fix

The minimal fix: **add a fallback root**. When the Node.js marker file search fails, fall back to a `.git` directory search (universal project root indicator), then fall back to `ctx.directory` (the file's directory) as final default.

```ts
export const YamlLS: Info = {
    id: "yaml-ls",
    extensions: [".yaml", ".yml"],
    root: NearestRoot([
        // try Node.js project markers first
        "package-lock.json", "bun.lockb", "bun.lock",
        "pnpm-lock.yaml", "yarn.lock",
        // fall back to universal project markers
        ".yamllint", "kustomization.yaml", "docker-compose.yaml",
        // final fallback: .git directory (any version-controlled project)
        ".git",
    ]),
    async spawn(root) {
        // ...
    },
}
```

`.git` is always a directory, so `NearestRoot`'s underlying `Filesystem.up` search will correctly find it. The function walks up until it hits the filesystem root, so it'll find the nearest `.git` automatically.

If even `.git` isn't found (bare files outside a repo), the caller should fall back to the file's directory. This requires a one-line change in `lsp/index.ts` where roots are resolved:

```ts
// Current: root = undefined → LSP never spawns
// Fixed: root = file's directory
const resolved = await server.root(file)
const root = resolved ?? path.dirname(file)
```

## Contribution Protocol

From `CONTRIBUTING.md`:
1. **Issue first.** File a bug report issue with clear reproduction steps.
2. **PR body MUST include `Fixes #XXXXX`** (or `Closes`, `Resolves`) — exact keyword, case-sensitive. The `Issue: https://...` format is silently ignored by the bot.
3. Target `dev` branch.
4. One commit per logical change.
5. Screenshot evidence of before/after for UI-visible bugs.

## Workaround (User-Side)

Until upstream fix is merged, users can disable the built-in `yaml-ls` and register a custom LSP with identical command and extensions. Custom LSPs receive `ctx.directory` as the default root, bypassing the project marker requirement entirely.

```jsonc
// In opencode.jsonc:
"lsp": {
  "yaml-ls": { "disabled": true },
  "yaml-anywhere": {
    "command": ["yaml-language-server", "--stdio"],
    "extensions": [".yaml", ".yml"]
  }
}
```
