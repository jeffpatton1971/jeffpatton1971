# Local AI Development Environment Guide
## Continue + Ollama + MCP + PowerShell + Windows

## Goal

Create a local AI coding environment that behaves as close to ChatGPT Codex as possible while:

- avoiding token costs
- staying local
- understanding repositories
- following architecture conventions
- supporting Windows and PowerShell correctly
- scaling across multiple local repositories
- minimizing hallucinations
- using MCP servers for external tools and context

## Recommended Architecture

```text
VS Code
+ Continue Extension
+ Ollama
+ qwen3-coder:30b
+ .continue/rules
+ .continueignore
+ MCP servers
```

Recommended MCP rollout:

```text
1. Context7
2. Filesystem
3. GitHub
```

Do not enable every MCP server at once. Add one, validate it, then add the next.

---

# 1. Install Prerequisites

## Install VS Code

Install Visual Studio Code.

Recommended extensions:

- Continue
- PowerShell
- C#
- YAML

## Install Ollama

Download Ollama from:

```text
https://ollama.com/download
```

Verify:

```powershell
ollama --version
```

Pull the main coding model:

```powershell
ollama pull qwen3-coder:30b
```

Pull a lightweight autocomplete model:

```powershell
ollama pull qwen3
```

Verify available models:

```powershell
ollama list
```

## Install Node.js

Node.js/npm are required for several MCP servers.

Download from:

```text
https://nodejs.org/
```

Verify:

```powershell
node --version
npm --version
npx --version
```

## Install Docker Desktop

Docker is required for the official GitHub MCP server.

Download from:

```text
https://www.docker.com/products/docker-desktop/
```

Verify Docker is running:

```powershell
docker version
```

---

# 2. VS Code Setup

Set PowerShell as the default terminal.

Open VS Code settings JSON and add:

```json
{
  "terminal.integrated.defaultProfile.windows": "PowerShell"
}
```

This helps reduce bash/Linux command assumptions such as:

- `cat`
- `grep`
- `sed`
- `awk`
- `rm`
- `cp`

---

# 3. Continue Main Config

Keep the main Continue config focused on models only. Put MCP server config in `.continue/mcpServers/`.

Recommended model config:

```yaml
name: Local Config
version: 1.0.0
schema: v1

models:
  - name: qwen-coder
    provider: ollama
    model: qwen3-coder:30b
    apiBase: http://localhost:11434
    roles:
      - chat
      - edit
      - apply
    capabilities:
      - tool_use
    defaultCompletionOptions:
      contextLength: 32768
      temperature: 0.2
      num_predict: 4096

  - name: qwen-autocomplete
    provider: ollama
    model: qwen3
    apiBase: http://localhost:11434
    roles:
      - autocomplete
    defaultCompletionOptions:
      contextLength: 4096
      temperature: 0.1
```

## Why `contextLength: 32768`?

Larger context is not always better locally.

Higher values can:

- increase RAM/VRAM usage
- slow responses
- increase context overflow
- worsen reasoning quality
- cause huge MCP tool results to overload the prompt

Try `65536` only after the 32768 setup is stable.

---

# 4. Repository Structure

Each repository should contain:

```text
.continue/
  rules/
  mcpServers/
.continueignore
AGENTS.md
```

Use repo-local config so each repository can have:

- its own filesystem MCP scope
- its own architecture rules
- its own build/test instructions
- its own ignored folders

---

# 5. MCP Server Setup

## Important MCP Rules

Do not:

- enable all MCP servers at once
- expose huge filesystem roots
- put secrets in YAML files
- read large files in full
- rely on MCP alone without rules

Recommended rollout:

```text
1. Context7
2. Filesystem
3. GitHub
```

Continue uses MCP servers in Agent mode.

---

## 5.1 Context7 MCP

### Purpose

Context7 provides documentation lookup for:

- SDKs
- frameworks
- Terraform providers
- GitHub Actions
- PowerShell/.NET libraries
- public package docs

### Optional global install

```powershell
npm install -g @upstash/context7-mcp
```

### Config file

Create:

```text
.continue/mcpServers/context7.yaml
```

Contents:

```yaml
name: Context7
version: 0.0.1
schema: v1

mcpServers:
  - name: context7
    command: npx
    args:
      - -y
      - "@upstash/context7-mcp"
```

### Manual validation

Run:

```powershell
npx -y @upstash/context7-mcp
```

If it starts and waits for MCP input without a package/download/auth error, it is likely working.

Use `Ctrl+C` to exit.

---

## 5.2 Filesystem MCP

### Purpose

Filesystem MCP provides:

- directory traversal
- repository inspection
- file reading
- safer file access than ad-hoc shell commands

### Scope carefully

Prefer scoping to the current repo.

Bad:

```text
C:\Users\Jeffrey\source
```

Better:

```text
C:\Users\Jeffrey\source\MyRepo
```

Broad filesystem access can cause the agent to wander.

### Optional global install

```powershell
npm install -g @modelcontextprotocol/server-filesystem
```

### Config file

Create:

```text
.continue/mcpServers/filesystem.yaml
```

Contents:

```yaml
name: Filesystem
version: 0.0.1
schema: v1

mcpServers:
  - name: filesystem
    command: npx
    args:
      - -y
      - "@modelcontextprotocol/server-filesystem"
      - "C:\\Users\\Jeffrey\\source\\MyRepo"
```

Replace the path with the actual repository path.

### Manual validation

```powershell
npx -y @modelcontextprotocol/server-filesystem "C:\Users\Jeffrey\source\MyRepo"
```

If it starts and waits for MCP input, it is likely working.

Use `Ctrl+C` to exit.

---

## 5.3 GitHub MCP

### Purpose

GitHub MCP provides access to:

- repositories
- issues
- pull requests
- commits
- workflows
- repository metadata

This helps the agent understand history and cross-repo context.

### Recommended approach

Use the official Docker image:

```text
ghcr.io/github/github-mcp-server
```

The official GitHub MCP server supports local Docker usage and requires Docker plus a GitHub Personal Access Token.

### Required prerequisites

- Docker Desktop installed and running
- GitHub Personal Access Token
- VS Code launched from a shell where the token exists

### Create a GitHub PAT

Create a GitHub Personal Access Token with only the permissions you are comfortable granting.

Common permissions/scopes depend on what you want the agent to do:

- repository read access
- issues read/write if needed
- pull requests read/write if needed
- workflow read access if needed
- organization read access if needed

Avoid over-scoping the token.

### Set the PAT before launching VS Code

In PowerShell:

```powershell
$env:GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxxxxxxxx"
code .
```

Important:

- VS Code inherits environment variables only at launch.
- If VS Code was already open, close every VS Code window first.
- Relaunch VS Code from the same PowerShell session.

### Pull the image

```powershell
docker pull ghcr.io/github/github-mcp-server
```

If this fails with auth-related GHCR issues, try:

```powershell
docker logout ghcr.io
docker pull ghcr.io/github/github-mcp-server
```

The image is public, but stale GHCR auth can interfere.

### Validate the server manually

Run:

```powershell
docker run -i --rm `
  -e GITHUB_PERSONAL_ACCESS_TOKEN `
  ghcr.io/github/github-mcp-server
```

Expected behavior:

- the container starts
- it waits for MCP stdio input
- it does not immediately exit
- it does not print authentication errors

Use `Ctrl+C` to exit.

### If the container exits immediately

Run it without `--rm` so you can inspect logs:

```powershell
docker run -i `
  --name github-mcp-test `
  -e GITHUB_PERSONAL_ACCESS_TOKEN `
  ghcr.io/github/github-mcp-server
```

Then inspect:

```powershell
docker logs github-mcp-test
docker rm github-mcp-test
```

### Continue config file

Create:

```text
.continue/mcpServers/github.yaml
```

Contents:

```yaml
name: GitHub
version: 0.0.1
schema: v1

mcpServers:
  - name: github
    command: docker
    args:
      - run
      - -i
      - --rm
      - -e
      - GITHUB_PERSONAL_ACCESS_TOKEN
      - ghcr.io/github/github-mcp-server
```

### Do not hardcode the PAT

Avoid this:

```yaml
env:
  GITHUB_PERSONAL_ACCESS_TOKEN: "ghp_xxxxx"
```

Prefer launching VS Code from PowerShell after setting the environment variable.

### Common GitHub MCP failures

#### `Connection closed`

Usually means the Docker process exited immediately.

Common causes:

- Docker Desktop is not running
- `GITHUB_PERSONAL_ACCESS_TOKEN` is not available to the Docker process
- VS Code was not launched from the shell where the token was set
- token is invalid/expired
- Docker image failed to pull/start
- corporate proxy/network blocked GitHub
- stale GHCR Docker auth
- server crashed during tool discovery

Fix order:

```powershell
docker version
docker pull ghcr.io/github/github-mcp-server
$env:GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxxxx"
docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

Then close VS Code and relaunch:

```powershell
code .
```

#### `Operation was aborted`

Usually means the MCP server took too long to start or tool discovery stalled.

Fixes:

- enable only GitHub MCP temporarily
- make sure Docker image is already pulled
- close and reopen VS Code
- disable Context7/filesystem temporarily
- confirm token is present before launch

---

# 6. .continueignore

Create:

```text
.continueignore
```

Recommended contents:

```text
bin/
obj/
node_modules/
packages/
dist/
build/
coverage/
.terraform/
terraform/
artifacts/
.vs/
.idea/
.git/
*.dll
*.exe
*.pdb
*.zip
*.nupkg
*.log
*.lock
*.generated.*
*.Designer.cs
*.g.cs
*.min.js
*.map
*.sarif
*.trx
TestResults/
```

This improves:

- context quality
- speed
- relevance
- local model performance

---

# 7. Continue Rules

Create:

```text
.continue/rules/
```

Recommended files:

```text
architecture.md
coding-standards.md
agent-workflow.md
powershell-environment.md
testing.md
build-and-ci.md
```

Optional topic-specific rules:

```text
terraform-rendering.md
github-actions.md
plugin-architecture.md
domain-models.md
api-patterns.md
documentation.md
```

---

## 7.1 PowerShell Environment Rule

Create:

```text
.continue/rules/powershell-environment.md
```

Contents:

```md
# PowerShell Environment

This repository is developed primarily on Windows using PowerShell.

## Command Rules

Prefer PowerShell commands instead of bash utilities.

Use:
- Get-Content instead of cat
- Get-ChildItem instead of ls
- Select-String instead of grep
- Copy-Item instead of cp
- Move-Item instead of mv
- Remove-Item instead of rm

## Shell Assumptions

- Do not assume bash is installed.
- Do not assume GNU utilities exist.
- Avoid Linux-specific shell syntax.
- Prefer PowerShell-compatible syntax.

## File Handling

- Do not read large files in full.
- Search before reading.
- Read only relevant sections.
```

---

## 7.2 Agent Workflow Rule

Create or update:

```text
.continue/rules/agent-workflow.md
```

Suggested contents:

```md
# Agent Workflow

## General Behavior

- Search and inspect before editing.
- Prefer small, focused changes.
- Do not invent file paths, APIs, classes, methods, or configuration names.
- Summarize intended changes before editing.
- Preserve existing project patterns.
- Update tests when behavior changes.
- Update documentation when user-facing behavior changes.
- Avoid generated/build output folders.
- Ask before large refactors.

## Tool Usage Policy

Use MCP tools before shell commands when available.

Use Context7 for:
- framework documentation
- SDK/API docs
- Terraform provider docs
- GitHub Actions docs

Use filesystem MCP for:
- repository inspection
- file reading
- directory traversal

Use GitHub MCP for:
- pull requests
- issues
- commits
- workflow history
- repository metadata

Rules strongly bias tool use, but they do not guarantee it.

## Large File Handling

- Do not read large files in full.
- Search before reading.
- Inspect file size and structure first.
- Read only relevant sections.
- For C# files, search for classes, methods, namespaces, and public types before reading full bodies.
- For JSON/YAML/XML, search for relevant keys first.
- Do not read generated files unless explicitly required.
```

---

# 8. PowerShell File Inspection Patterns

Use these instead of Linux commands.

## File size

```powershell
Get-Item .\file.cs | Select-Object FullName, Length
```

## Read top of file

```powershell
Get-Content .\file.cs -TotalCount 100
```

## Read partial section

```powershell
Get-Content .\file.cs | Select-Object -Skip 200 -First 100
```

## Search one file

```powershell
Select-String -Path .\file.cs -Pattern "SomeClass"
```

## Search recursively

```powershell
Get-ChildItem -Recurse -Include *.cs | Select-String -Pattern "SomeMethod"
```

---

# 9. Best Prompting Workflow

## Step 1: Explore first

```text
Inspect the repository.
Identify the relevant files.
Summarize the current implementation.
Do not make changes yet.
Use PowerShell-compatible commands.
Do not read large files in full.
Use MCP tools where available.
```

## Step 2: Implement second

```text
Now implement the smallest safe change.
Follow existing patterns.
Update tests if needed.
Avoid unnecessary edits.
Do not read large files in full.
```

---

# 10. AI Bootstrap Prompt for New Repos

Use this prompt inside a new repository to generate initial Continue rules.

```text
Create Continue rules for this repository.

Goal:
Generate a `.continue/rules/` folder containing concise, repo-specific Markdown rule files that help Continue Agent mode work effectively with a local Ollama model.

Do not modify application code. Only create or update files under `.continue/rules/`, `.continueignore`, and optionally `AGENTS.md`.

First inspect the repo:
- README files
- docs/
- src/
- tests/
- build files
- GitHub workflows
- solution/project files
- package/module manifests
- existing conventions

Then create these files as applicable:

1. `.continue/rules/architecture.md`
Explain the repository purpose, major components, folder layout, ownership boundaries, and where common changes should be made.

2. `.continue/rules/coding-standards.md`
Capture naming conventions, language/framework patterns, dependency rules, error handling, logging, formatting, and things the agent must not do.

3. `.continue/rules/testing.md`
Explain test project layout, test frameworks, how to run tests, naming conventions, fixture patterns, and expectations for adding/updating tests.

4. `.continue/rules/build-and-ci.md`
Document build commands, packaging commands, CI workflow behavior, release/versioning conventions, and any required local prerequisites.

5. `.continue/rules/agent-workflow.md`
Add general instructions:
- Search/read before editing.
- Prefer small focused changes.
- Do not invent file paths, APIs, classes, methods, or configuration names.
- Summarize intended changes before editing.
- Preserve existing patterns.
- Update docs/tests when behavior changes.
- Avoid generated/build output folders.
- Ask before large refactors.
- Use MCP tools when available.
- Do not read large files in full.

6. `.continue/rules/powershell-environment.md`
Configure PowerShell-specific guidance:
- Prefer PowerShell commands.
- Avoid Linux/bash assumptions.
- Avoid cat/grep/sed/awk.
- Search before reading large files.

7. Create additional topic-specific rule files only if the repo clearly needs them:
- powershell.md
- terraform-rendering.md
- github-actions.md
- api-patterns.md
- plugin-architecture.md
- domain-models.md
- documentation.md

Rules:
- Keep files concise.
- Prefer bullets over long prose.
- Include concrete repo-specific examples.
- Avoid huge documentation dumps.
- Do not include secrets.
- Mark uncertain areas with:
  "Verify before changing."

Also create `.continueignore` if it does not exist.

After creating the files:
- summarize conventions discovered,
- summarize assumptions,
- identify unclear architecture areas,
- recommend future rule files.
```

---

# 11. BAT/vNext Additions

For BAT/vNext repositories, add rule files such as:

```text
terraform-rendering.md
plugin-architecture.md
domain-models.md
github-actions.md
```

Important BAT/vNext themes:

- provider vs renderer separation
- defaulting and validation boundaries
- plugin registration patterns
- resource model conventions
- Terraform output rules
- PowerShell module/cmdlet conventions
- tests as the source of truth

---

# 12. Verification

## Verify Ollama is being used

Run:

```powershell
ollama ps
```

Then send a Continue prompt. You should see:

```text
qwen3-coder:30b
```

## Verify MCP servers are running

For Node-based MCP servers:

```powershell
Get-Process node
```

For GitHub MCP:

```powershell
docker ps
```

## Verify tool usage

Prompt:

```text
Use filesystem MCP to inspect this repository.
Use Context7 to retrieve documentation relevant to the primary framework used here.
Summarize the architecture before making any edits.
Do not use bash commands.
```

You should see tool activity in Continue.

---

# 13. Common Problems

## `Tool cat not found`

Cause:

- model assumed Linux/bash tooling

Fix:

- PowerShell rule file
- PowerShell default terminal
- explicit prompt guidance
- use filesystem MCP where possible

## `Message exceeds context limit`

Cause:

- full-file reads
- huge MCP results
- generated files
- missing `.continueignore`

Fix:

- partial reads
- search-first workflow
- aggressive `.continueignore`
- scoped filesystem MCP

## `Failed to connect to MCP`

Cause:

- server not installed
- server startup too slow
- Docker unavailable
- token missing
- network/proxy issue
- too many MCPs starting at once

Fix:

- enable one MCP at a time
- validate manually
- restart VS Code
- check logs

## GitHub MCP `Connection closed`

Cause:

- Docker container exited immediately

Fix:

```powershell
docker version
docker pull ghcr.io/github/github-mcp-server
$env:GITHUB_PERSONAL_ACCESS_TOKEN = "ghp_xxxxx"
docker run -i --rm -e GITHUB_PERSONAL_ACCESS_TOKEN ghcr.io/github/github-mcp-server
```

Then relaunch VS Code from that same PowerShell session:

```powershell
code .
```

---

# 14. Recommended Stable Setup

Start with:

```text
Continue
+ Ollama
+ qwen3-coder:30b
+ repo rules
+ .continueignore
+ Context7 MCP
+ filesystem MCP scoped to current repo
```

Add GitHub MCP only after the first two MCPs are stable.

---

# 15. Long-Term Improvement

Eventually consider a custom BAT MCP server.

Possible capabilities:

- ADR retrieval
- renderer examples
- plugin conventions
- schema discovery
- Terraform patterns
- golden YAML retrieval
- pipeline examples

This would provide:

- cross-repo architecture memory
- shared conventions
- significantly better grounding
- better local agent performance
- 
