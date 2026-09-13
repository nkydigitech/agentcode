# Security-First Agentic AI on AWS — Bedrock AgentCore (SEC307)

**Cloud & DevOps Engineer | I turn manual, 3 AM-breaking deployments into 1-min automated pipelines with AWS + Ansible + Terraform | Featured: 15-Module Ansible Lab with real terminal**

A production-pattern AI agent for AnyCompany Retail — a Strands agent running Claude Opus 4.6 on **AWS Bedrock AgentCore**, where every tool it calls is secured with a different authentication pattern, every tool listing is authorized per-request, and the agent is *architected to decline rather than guess*.

Built from the AWS re:Invent 2025 **SEC307** workshop (Bridging the Gap Between Agentic AI and Security Best Practices) — deployed, tested, and documented with 64 real AWS console screenshots.

🔗 [LinkedIn — Nkechi Ahanonye](https://www.linkedin.com/in/nkechiahanonye)

---

## Why This Project Matters

Most AI agent demos hand the agent a wide-open API key and hope for the best. This build answers the question every security team is asking right now: *how do you give an AI agent powerful tools without giving it the keys to the kingdom?*

The answer here is defense in depth, applied to agentic AI:

1. **Least-privilege tool access** — each MCP tool behind a gateway with its own auth model
2. **Zero embedded secrets** — OAuth credentials live in the AgentCore Identity vault and are exchanged for tokens at runtime (M2M)
3. **Per-request tool authorization** — Amazon Verified Permissions filters the tool list for what the *calling principal* is allowed to use
4. **Anti-hallucination grounding** — a system prompt with a hard rule: relay tool results or decline. Never invent policies, prices, or terms
5. **Observability** — CloudWatch log delivery for every agent session

## The Three Tool-Security Patterns

| Activity | Tool | Auth Pattern | Where Credentials Live |
|---|---|---|---|
| Activity 2 | AnyCompany ToS Tool (refund/delivery/payment terms) | **SigV4 + IAM** inbound auth on AgentCore Gateway | IAM execution role (no keys at all) |
| Activity 3a | AnyCompany Sales & Product Reviews Tool | **OAuth2 M2M** via Cognito + AgentCore Identity | AgentCore Identity **vault** — exchanged for tokens at runtime, never in code |
| Activity 5 | All tools | **Amazon Verified Permissions (AVP)** | Policy store — per-principal, per-tool authorization |

The AVP layer is the standout: when enabled, the agent asks the policy store *on every request* which tools this principal may `AccessTool` against. An unauthorized caller gets an agent that simply doesn't show — let alone call — the tools they have no business touching. Failed authorizations are logged, and the agent skips unauthorized tool groups gracefully.

## Anti-Hallucination by Design

The system prompt enforces one critical grounding rule: the agent has **no built-in knowledge** of AnyCompany's business data. Terms of service, prices, inventory, delivery times — none of it. It may only state what a tool returned *in the current conversation*. If no tool has the answer, it responds with exactly one decline sentence and moves on.

No partial answers. No "typically retailers do this...". No disclaimers bolted onto guesses. Either relay the tool result, or decline.

## Architecture

```
User → Cognito JWT (custom authorizer + header allowlist)
     → Bedrock AgentCore Runtime (container, ECR, CodeBuild pipeline)
     → Strands Agent (Claude Opus 4.6 on Bedrock)
        ├─ ToS Tool        → AgentCore Gateway  → SigV4/IAM
        ├─ Sales Tool      → AgentCore Gateway  → OAuth2 M2M (Identity vault)
        └─ AVP Policy Store → per-request tool filtering
     → CloudWatch (session logs, tool decisions, auth failures)
```

Runtime details (`.bedrock_agentcore.yaml`): container deployment, linux/arm64, CodeBuild build pipeline, public network mode, us-east-1.

## Proof of Deployment

Every step of this build is documented in `screenshots/` — 64 real AWS console captures, including:

1. AgentCore runtime deployed + agent details
2. Cognito identity corp pool + app clients
3. Gateways and targets for both MCP tools
4. IAM roles and trust relationships
5. Secrets Manager (where secrets do — and don't — live)
6. CloudFormation stacks: bootstrap, direct-targets, standalone MCP server
7. CloudWatch log events and application logs
8. The sales operations assistant frontend working end to end — successful tool-backed answers

## Repo Structure

```
agent.py                    # Strands agent: tool-group registry, AVP gate, grounding prompt
.bedrock_agentcore.yaml     # AgentCore runtime config (roles, ECR, region, protocol)
deploy-agentcore-runtime.sh # Full deploy workflow with Cognito JWT authorizer
launchAgent.sh              # Local + remote launch flow
activity2_gen.sh            # Registers ToS tool group (SigV4 transport)
activity3a/3b/3c_gen.sh     # Registers Sales tool group (Identity vault OAuth2 M2M)
activity4_gen.sh            # Standalone MCP server connection
activity5_gen.sh            # Enables AVP dynamic tool filtering
requirements.txt            # bedrock-agentcore, strands-agents, mcp (pinned)
screenshots/                # 64 deployment & test captures
```

## How to Run

```bash
# 1. Configure your AWS credentials + region (us-east-1)
aws configure

# 2. Install dependencies
pip install -r requirements.txt

# 3. Deploy the AgentCore runtime (Cognito JWT authorizer included)
chmod +x deploy-agentcore-runtime.sh
./deploy-agentcore-runtime.sh

# 4. Launch the agent
./launchAgent.sh
```

Each `activity*_gen.sh` script updates `agent.py` with that activity's MCP client configuration — run them in order to layer in the security patterns one at a time, exactly as the workshop intended.

---

Built by **Nkechi Anna Ahanonye** · [LinkedIn](https://www.linkedin.com/in/nkechiahanonye) · [GitHub](https://github.com/nkydigitech)

*Agentic AI with the security posture your CISO actually wants.*
