# Pattern: Skill-Driven Agents with Files API + Remote MCP

## Problem

A production AI assistant for an enterprise client needs three things at once:

1. **Persistent procedural expertise** — the model must follow the client's operating procedures the same way on every call, not depend on a 4000-token system prompt being copy-pasted into every request by every caller.
2. **Tenant-specific reference documents** — manuals, SOPs, regulatory PDFs that change quarterly. These need to be attachable by ID, not re-uploaded on every conversation.
3. **Access to the client's operational systems** — work order systems, asset databases, scheduling APIs. Each one is a different REST API with different auth.

The traditional answer is: build a custom orchestration layer. Embed procedures in giant system prompts. Re-upload PDFs as base64 in every request. Write a JSON-RPC client for each backend system, deploy it as a sidecar, register it as a tool. By the time the orchestration is "done," you have a fragile multi-service stack to maintain per client.

Anthropic's May 2026 beta surface collapses this into three composable primitives that ship with the Messages API. The composition pattern below is what we use to onboard a new enterprise client in days instead of weeks.

---

## Solution

### Primitives in Play

| Primitive | What It Replaces | Beta Status (May 2026) |
|---|---|---|
| Skills | Per-request system prompt boilerplate | Beta |
| Files API | Re-uploading PDFs every conversation | Beta |
| Remote MCP Connector | Custom MCP client code per tool server | Beta |

These primitives compose. A single `messages.create` call can invoke a Skill, reference Files by ID, and call into a remote MCP server. The model does the orchestration; your code does the routing.

### Step 1: Define a Skill for the Client's Procedures

A Skill is a named, versioned instruction package stored on Anthropic's side. You upload it once. Every request that names the Skill gets the same behavior, deterministically.

```typescript
// scripts/install-client-skill.ts
// Run once per client onboarding.

import Anthropic from '@anthropic-ai/sdk';

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
  defaultHeaders: {
    'anthropic-beta': 'skills-2026-05-01',
  },
});

export async function installClientProcedureSkill(clientOrgId: string) {
  return anthropic.beta.skills.create({
    name: `${clientOrgId}-field-operations`,
    description:
      'Field operations procedural knowledge for ' +
      `${clientOrgId}. Used by GarageBuddy and DigBuddy ` +
      'when answering on-site diagnostic and compliance queries.',
    instructions: `
You are the field operations assistant for ${clientOrgId}.

Procedural rules — apply on every response:
1. If the worker asks about a procedure, retrieve the relevant SOP via the
   attached files. Cite the file and section in your answer.
2. If a procedure involves a regulated activity (excavation, gas line work,
   pressurized chemical handling), include the required safety prerequisites
   before describing the procedure.
3. If you cannot find the procedure in attached files, say so explicitly.
   Do not infer procedures from training data — for this client, that is a
   compliance violation.
4. When the worker reports a fault code, use the diagnostic MCP tool to look
   it up in the client's asset history before recommending action.

Tone — terse, field-appropriate. No marketing language. No emojis.
    `.trim(),
    model: 'claude-sonnet-4-6',
  });
}
```

The Skill ID returned (e.g. `skill_01abc...`) is what you reference in every subsequent request. The instructions live on Anthropic's side and are not re-sent. This saves tokens, but more importantly it means the procedural contract for this client is **one thing in one place**, not duplicated across every caller.

### Step 2: Attach Reference Documents via Files API

When the client uploads a new procedure manual, you upload it to the Files API once. You get back a `file_id` you can reference from any subsequent request.

```typescript
// lib/client-files.ts

import Anthropic from '@anthropic-ai/sdk';
import { createReadStream } from 'fs';
import { createClient } from '@supabase/supabase-js';

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
  defaultHeaders: {
    'anthropic-beta': 'files-api-2025-04-14',
  },
});

const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!,
);

export async function attachClientManual(
  clientOrgId: string,
  localPath: string,
  category: 'sop' | 'regulatory' | 'asset-history',
) {
  // 1. Upload to Anthropic Files API — returns a persistent file_id
  const file = await anthropic.beta.files.upload({
    file: createReadStream(localPath),
  });

  // 2. Track the file_id in our own DB, scoped to client + category.
  //    RLS ensures only this client's requests can reference these file_ids.
  await supabase.from('client_files').insert({
    client_org_id: clientOrgId,
    file_id: file.id,
    category,
    source_path: localPath,
  });

  return file.id;
}

export async function resolveClientFiles(
  clientOrgId: string,
  categories: Array<'sop' | 'regulatory' | 'asset-history'>,
): Promise<string[]> {
  const { data, error } = await supabase
    .from('client_files')
    .select('file_id')
    .eq('client_org_id', clientOrgId)
    .in('category', categories);

  if (error) throw new Error(`Failed to resolve files: ${error.message}`);
  return (data ?? []).map((row) => row.file_id);
}
```

Files persist server-side. You never re-upload a 60-page manual on every conversation turn. The `client_files` table is your tenant boundary — RLS keeps client A from ever resolving client B's file IDs.

### Step 3: Wire In a Remote MCP Server

The client has a work-order system at `https://wo.acme-corp.internal/mcp` that exposes MCP-compatible endpoints for asset lookup, dispatch creation, and history queries. Pre-May 2026, you would write a custom MCP client, deploy it, and register it as a tool. Now you point Claude at the URL.

```typescript
// lib/client-mcp.ts

export interface ClientMCPConfig {
  url: string;
  name: string;
  authHeader: string;       // injected per-request from secret store
  toolsAllowed?: string[];  // optional allowlist
}

export async function loadClientMCP(
  clientOrgId: string,
): Promise<ClientMCPConfig> {
  // Pull from your secret store, scoped per tenant.
  // Never hardcode. Never log the auth header.
  const config = await fetchSecret(`mcp:${clientOrgId}`);
  return {
    url: config.url,
    name: `${clientOrgId}-operations`,
    authHeader: `Bearer ${config.token}`,
    toolsAllowed: config.tools_allowed,
  };
}
```

### Step 4: The Composed Request

This is where the three primitives compose into one call.

```typescript
// app/api/field-assistant/route.ts

import Anthropic from '@anthropic-ai/sdk';
import { resolveClientFiles } from '@/lib/client-files';
import { loadClientMCP } from '@/lib/client-mcp';
import { getClientSkillId } from '@/lib/client-config';

const anthropic = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
  defaultHeaders: {
    'anthropic-beta':
      'skills-2026-05-01,files-api-2025-04-14,mcp-client-2025-04-04',
  },
});

export async function POST(req: Request) {
  const { query, clientOrgId, conversationHistory } = await req.json();

  // 1. Resolve tenant context — skill, files, MCP — all in parallel.
  const [skillId, fileIds, mcpConfig] = await Promise.all([
    getClientSkillId(clientOrgId),
    resolveClientFiles(clientOrgId, ['sop', 'regulatory']),
    loadClientMCP(clientOrgId),
  ]);

  // 2. One call. Claude orchestrates skill + files + MCP tools.
  const response = await anthropic.beta.messages.create({
    model: 'claude-sonnet-4-6',
    max_tokens: 2048,

    // Skill replaces the system prompt for procedural knowledge
    skill_id: skillId,

    // Files are referenced by ID — no re-upload
    messages: [
      ...conversationHistory,
      {
        role: 'user',
        content: [
          ...fileIds.map((id) => ({
            type: 'document' as const,
            source: { type: 'file' as const, file_id: id },
            citations: { enabled: true },
          })),
          { type: 'text' as const, text: query },
        ],
      },
    ],

    // Remote MCP — Claude calls the URL directly. No client glue.
    mcp_servers: [
      {
        type: 'url',
        url: mcpConfig.url,
        name: mcpConfig.name,
        authorization_token: mcpConfig.authHeader,
        ...(mcpConfig.toolsAllowed && {
          tool_configuration: { allowed_tools: mcpConfig.toolsAllowed },
        }),
      },
    ],
  });

  return Response.json(response);
}
```

The model reads the Skill instructions, decides whether to consult the attached files, and calls into the client's MCP server when the procedure requires looking up live asset data. Your code does none of that orchestration. You routed the right tenant context in, and you return the response.

### Step 5: Prompt Caching for Repeated Files

Files referenced by `file_id` are eligible for prompt caching when the same file is referenced across requests. For a field assistant where the same SOP gets queried hundreds of times an hour during a shift, this is the difference between a $40 day and a $400 day.

```typescript
// In the document block above, mark the file for caching:
{
  type: 'document',
  source: { type: 'file', file_id: id },
  citations: { enabled: true },
  cache_control: { type: 'ephemeral' },  // 5-minute TTL cache
}
```

Cache the procedural files. Don't cache the user's query. Don't cache the MCP response. The cost reduction is proportional to file size and request frequency — for our deployments the steady-state hit rate sits around 80% during active shifts.

---

## Why It Works

- **Each primitive does one thing.** Skills hold procedural knowledge. Files hold reference documents. MCP holds external system access. None of them try to do the others' job, so each one is sharp.
- **The orchestration is the model's problem, not yours.** Claude decides when to consult a file and when to call an MCP tool based on the Skill's instructions. You don't write a planner. You don't write a tool-routing layer. You ship the tenant context and read the response.
- **Tenant isolation lives at three layers.** Per-tenant Skill ID (instructions can't leak), per-tenant file_ids resolved through RLS (documents can't leak), per-tenant MCP URL with per-tenant auth (system access can't leak). Defense in depth means no single misconfigured row leaks a client's operating data.
- **Onboarding a new client is config, not code.** Install a Skill. Upload some files. Register an MCP URL in the secrets vault. The route handler doesn't change. The same TypeScript serves client 1 and client 50.

---

## Gotchas

**1. Skills are versioned but not auto-rolled-back.**
When you update a Skill, the new version is live immediately for every request that uses that Skill ID. There is no canary, no traffic shifting. If your new procedural rules are wrong, every active worker gets the wrong answer the next time they ask. Treat Skill updates like database migrations: version-controlled in git, reviewed before deployment, and tested against a regression set of representative queries.

**2. Remote MCP servers must be idempotent for retries.**
The Claude API will retry MCP calls on transient failures. If your MCP server exposes a `create_work_order` tool and your retry creates a duplicate work order, you have a production data quality problem. Either make the tool idempotent (accept a client-generated request ID and dedupe), or expose only read operations through MCP and require a separate confirmation step for writes.

**3. Files API has per-account storage limits.**
You cannot use it as a substitute for your own document store. Upload only the documents that are referenced in active prompts. When a client churns or rotates their SOPs, delete the old `file_id` entries — both from the Files API and from your tracking table. Otherwise you accumulate orphaned uploads that count against your storage limit and surface in nothing.

**4. Beta headers can change.**
`anthropic-beta: skills-2026-05-01` is the May 2026 header. When Skills GAs, the header changes or disappears. Pin the header in a single constants file, not scattered across route handlers, so a beta GA is a one-line change. Subscribe to the Anthropic release notes feed and treat header changes as P1 work, not whenever-cleanup.

---

## Real-World Note

This composition is how we onboard new enterprise clients onto FieldBuddy. The first client took a week of custom orchestration code. The third client took an afternoon: install the Skill, upload the SOPs, register the MCP URL. The route handler hasn't been touched since the second client.

The job that used to be "write a bespoke agent for each customer" is now "configure tenant context." That is the difference Anthropic's May 2026 surface makes for a small team supporting many enterprise deployments — the per-tenant labor cost drops from week-scale to hour-scale, and the surface area of code that can break in production shrinks to one route handler.
