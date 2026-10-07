# Claude Code Setup

How Claude Code is configured for this project: a `CLAUDE.md` with working rules, and MCP servers that give it access to AWS, Terraform and pricing data.

## CLAUDE.md

`CLAUDE.md` at the repository root is read automatically at the start of every session. This project's version puts Claude in a safe execution mode:

```
You are operating in safe execution mode.

Before executing any command:
- Before taking any action, briefly explain what you're about to do in 1-2 simple sentences
- Use plain language, avoid jargon
- Say WHY, not just WHAT
- Then proceed with the action

Always prefer clear reasoning before action.
```

Every command is explained before it runs, which matters when the session can touch live AWS infrastructure. Add project rules the same way, for example:

```markdown
# Always use the ecommerce namespace unless told otherwise
# Never run terraform apply without showing the plan first
# Branch naming: feature/<name>, fix/<name>
```

A `CLAUDE.md` in a subdirectory is read when Claude works in that directory.

## MCP servers

MCP (Model Context Protocol) servers run as background processes and expose extra tools to Claude.

| Server | Purpose | Example prompt |
|--------|---------|----------------|
| `awslabs.eks-mcp-server` | Inspect EKS clusters and Kubernetes resources: pods, events, logs, manifests | "Why is the gateway pod crashing?" |
| `awslabs.terraform-mcp-server` | Run Terraform commands, search provider documentation, run Checkov scans | "Run terraform plan in the infrastructure repo" |
| `awslabs.aws-pricing-mcp-server` | Live AWS pricing lookups and cost estimates | "What would a second m7i-flex.large node cost per month?" |

## Setup

### 1. Install Claude Code

```bash
npm install -g @anthropic-ai/claude-code
claude --version
claude            # first run opens a browser to sign in
```

### 2. Configure AWS credentials

```bash
aws configure
aws sts get-caller-identity
```

If the second command fails, the AWS MCP servers cannot connect. For SSO or named profiles, pass `-e AWS_PROFILE=<name>` when registering a server.

### 3. Install uv

`uvx` launches the MCP servers on demand.

```bash
brew install uv       # macOS; or: pip install uv
uvx --version
```

### 4. Register the servers

```bash
claude mcp add eks --scope user \
  -e AWS_REGION=us-east-1 -e FASTMCP_LOG_LEVEL=ERROR \
  -- uvx awslabs.eks-mcp-server@latest

claude mcp add terraform --scope user \
  -e FASTMCP_LOG_LEVEL=ERROR \
  -- uvx awslabs.terraform-mcp-server@latest

claude mcp add aws-pricing --scope user \
  -e AWS_REGION=us-east-1 -e FASTMCP_LOG_LEVEL=ERROR \
  -- uvx awslabs.aws-pricing-mcp-server@latest
```

`--scope` accepts `local` (default), `user` (all your projects) and `project` (writes `.mcp.json` for the team). Replace `us-east-1` with your region. Check the server documentation for flags that enable write access to the cluster.

### 5. Verify

```bash
claude mcp list
```

Inside a session, `/mcp` shows the same status.

| Problem | Fix |
|---------|-----|
| Server shows as failed | Run `aws sts get-caller-identity` to confirm credentials |
| Wrong-region results | Change `AWS_REGION` with `claude mcp remove <name>` and add it again |
| `uvx: command not found` | Install uv (step 3) |
| First connection times out | `uvx` downloads the server on first use; retry after about 30 seconds |

## Which server is used for what

| Task | Server |
|------|--------|
| Check pod logs and health | `eks` |
| Apply Kubernetes manifests | `eks` |
| Run `terraform plan` / `apply` | `terraform` |
| Look up provider resources | `terraform` |
| Estimate infrastructure cost | `aws-pricing` |
