# Applied Claude Patterns

Production patterns for building Claude-powered operational tooling at scale.

These are extracted from real deployments — AI assistants serving enterprise operational teams, multi-tenant platforms deployed across enterprise clients, and consumer mobile apps built on the same backend. Sanitized for public use.

## Who This Is For

Engineers building Claude into production systems, not demos. Every pattern here has been stress-tested against real operational workflows across regulated industries. The constraints are different from consumer AI — uptime matters, citations matter, tenant isolation matters, and hallucinated answers are a liability, not just an annoyance.

## Patterns

| Pattern | Problem Solved |
|---|---|
| [Universal Streaming Endpoint](patterns/universal-endpoint.md) | One backend serving N AI products with zero code duplication |
| [RAG with Native Citations](patterns/citations-rag.md) | Grounded answers with traceable sources — no hallucinated procedures |
| [Config-Driven Multi-Tenancy](patterns/multi-tenant-config.md) | Single codebase, N clients, isolated and branded per deployment |
| [Cross-Tenant Isolation Testing](patterns/cross-tenant-isolation.md) | Prove data never leaks between clients in CI |
| [17 Prompt Engineering Patterns](patterns/prompt-engineering-patterns.md) | Full system prompt architecture — from Persona to Few-Shot, all with production examples |
| [Skills + Files API + Remote MCP](patterns/skills-files-remote-mcp.md) | Onboard a new enterprise client in an afternoon — composing the May 2026 Anthropic beta surface |
| [Agent Security Spine](patterns/agent-security-spine.md) | Six default failure modes in agent frameworks, and the OS/network boundaries that actually contain them |

## Stack

These patterns are implemented in TypeScript on Vercel serverless functions. The RAG patterns use Supabase + pgvector. The isolation tests run in Vitest. Swap the infrastructure layer and the patterns hold.

## Dev Setup

If you're working with these patterns in Claude Code and your stack touches Google Cloud, install the official GCP skills for better context on current API syntax:

```bash
npx skills install github.com/google/skills
```

Recommended: `bigquery-basics`, `cloud-run-basics`, `google-cloud-recipe-auth`, and the three WAF pillars (security, reliability, cost-optimization). Install globally + symlink so updates pull automatically.

## Author

Jeremy Odam — [jeremyodam.com](https://jeremyodam.com)  
25 years in regulated utility operations. Building AI-augmented operational tooling.
