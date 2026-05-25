# Issue Creation Instructions

## Prerequisites

```bash
cd ~/repos/pngdeity/opencode
gh auth status   # must be authenticated as pngdeity
```

## Issue Content

### Title

```
YAML LSP (yaml-ls) fails to activate in non-Node.js projects — root detection relies on package manager lock files
```

### Body

```markdown
## Description

The built-in `yaml-ls` LSP server only activates for `.yml`/`.yaml` files when a Node.js package manager lock file (`package-lock.json`, `bun.lockb`, `bun.lock`, `pnpm-lock.yaml`, or `yarn.lock`) exists in the directory hierarchy. In any project that doesn't use one of these package managers — Kubernetes manifests, GitHub Actions workflows, Docker Compose, Ansible playbooks, CI/CD configs, APM manifests, OpenAPI specs — the YAML LSP silently fails to spawn.

## Steps to reproduce

1. Create a directory with a `.yml` file and no Node.js lock files:
   ```bash
   mkdir /tmp/yaml-test
   echo 'key: value' > /tmp/yaml-test/test.yml
   ```
2. Open `/tmp/yaml-test/test.yml` in OpenCode.
3. Observe: `yaml-ls` never starts. No diagnostics appear. No schema validation.
4. Repeat with `package-lock.json` present in the same directory — `yaml-ls` activates normally.

## Expected behavior

The YAML LSP should activate for `.yml` and `.yaml` files regardless of which package manager the project uses. YAML is a universal configuration format, not a Node.js-specific one.

## Root cause

`packages/opencode/src/lsp/server.ts` (~line 1399 on `dev`):

```ts
export const YamlLS: Info = {
    id: "yaml-ls",
    extensions: [".yaml", ".yml"],
    root: NearestRoot(["package-lock.json", "bun.lockb", "bun.lock",
                        "pnpm-lock.yaml", "yarn.lock"]),
    // ...
}
```

The `root` function uses `NearestRoot` with only Node.js lock file markers. When none are found in the directory tree, `root` returns `undefined` and the LSP never spawns. There is no fallback.

## Comparison with other LSPs

Other LSPs use language-appropriate root markers:
- `gopls` → `go.mod`
- `pyright` → `pyproject.toml`, `setup.py`
- `terraform` → `.terraform`, `.terragrunt`
- `lua-ls` → `.luarc.json`, `.luacheckrc`
- `bash` → `ctx.directory` (no marker needed)

`yaml-ls` is the only LSP with a root detection that's unrelated to the language it serves.

## Prior related reports

- **#7842** — same symptoms, auto-closed after 90 days
- **PR #6986** — fix submitted (added `.yamllint`, `kustomization.yaml`, `docker-compose.yaml` as root markers) but auto-closed due to missing `Fixes #XXXXX` keyword in PR body
- **#18694** — TypeScript LSP fails in monorepos, same restrictive root pattern

## Proposed fix

Add a fallback root chain:

```ts
root: NearestRoot([
    "package-lock.json", "bun.lockb", "bun.lock",
    "pnpm-lock.yaml", "yarn.lock",
    ".yamllint", "kustomization.yaml", "docker-compose.yaml",
    ".git",   // universal project root
])
```

And in the spawn resolution (in `lsp/index.ts`), fall back to the file's directory when no root is found:

```ts
const resolved = await server.root(file)
const root = resolved ?? path.dirname(file)
```

## Environment

- OpenCode version: 1.15.10 (AUR)
- OS: Arch Linux
- yaml-language-server: 1.22.0 (installed via pacman)
```

### Labels

```
bug, lsp
```

### Issue creation command

```bash
cd ~/repos/pngdeity/opencode
gh issue create \
  --repo anomalyco/opencode \
  --title "YAML LSP (yaml-ls) fails to activate in non-Node.js projects — root detection relies on package manager lock files" \
  --body "$(cat docs/yaml-lsp-issue-body.md)" \
  --label "bug,lsp"
```

Alternatively, save the body to a file and reference it:

```bash
# First, write the body content to a file
cat > /tmp/yaml-lsp-issue-body.md << 'ENDOFBODY'
<paste body content above>
ENDOFBODY

# Then create the issue
gh issue create \
  --repo anomalyco/opencode \
  --title "YAML LSP (yaml-ls) fails to activate in non-Node.js projects — root detection relies on package manager lock files" \
  --body "$(cat /tmp/yaml-lsp-issue-body.md)" \
  --label "bug,lsp"
```

### After issue creation

1. Note the issue number (e.g., `#XXXXX`).
2. Use it in the PR body as `Fixes #XXXXX` — exact format, case-sensitive.
3. The `Fixes` keyword is required by the issue-link bot. `Issue: https://...` format is silently ignored.
4. Target the `dev` branch for the PR.
