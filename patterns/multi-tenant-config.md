# Pattern: Config-Driven Multi-Tenancy

## Problem

You have one AI platform and multiple enterprise clients. Each client needs their own branding, their own system prompt tailored to their operations, their own document corpus for RAG, and their own set of enabled AI modules. Client A is a utility company running FieldBuddy and SafetyBuddy. Client B is a construction firm that only needs ProjectBuddy. Client C needs all of it with a red color scheme and their logo.

The naive solution is a separate deployment per client. That path leads to N Vercel projects, N Supabase instances, N deploy pipelines, N places for the same bug to live undetected until it surfaces in the one client you care about most. It also means your client onboarding process involves a DevOps engineer and a 45-minute window instead of a 10-minute script.

The correct architecture: one codebase, one deployment, N clients configured entirely through data. Onboarding a new client means inserting rows, not deploying infrastructure. Fixing a bug means one commit, not N hotfixes. Upgrading the model means changing one default, not chasing N environment variables across N projects.

---

## Solution

### Schema

```sql
-- clients: one row per enterprise client
CREATE TABLE clients (
  id            uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id        text UNIQUE NOT NULL,     -- slug used in API headers and RLS
  name          text NOT NULL,
  primary_color text DEFAULT '#2563eb',
  logo_url      text,
  active        boolean DEFAULT true,
  plan          text DEFAULT 'standard',  -- 'standard' | 'enterprise' | 'pilot'
  created_at    timestamptz DEFAULT now(),
  updated_at    timestamptz DEFAULT now()
);

-- module_configs: which AI modules are enabled per client, and how
CREATE TABLE module_configs (
  id              uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  client_org_id   text NOT NULL REFERENCES clients(org_id) ON DELETE CASCADE,
  module          text NOT NULL,            -- 'field-ops' | 'safety' | 'fleet' | etc.
  system_prompt   text NOT NULL,
  model           text DEFAULT 'claude-sonnet-4-5',
  max_tokens      int  DEFAULT 1024,
  temperature     float DEFAULT 0.2,
  rag_enabled     boolean DEFAULT false,
  rag_namespace   text,                     -- scopes pgvector retrieval per client+module
  enabled         boolean DEFAULT true,
  created_at      timestamptz DEFAULT now(),
  UNIQUE(client_org_id, module)             -- one config per module per client
);

-- Row-level security — every table that holds client data gets this
ALTER TABLE clients ENABLE ROW LEVEL SECURITY;
ALTER TABLE module_configs ENABLE ROW LEVEL SECURITY;

-- RLS policy reads org_id from session config set at request time
-- The application sets this before any query; the DB enforces it
CREATE POLICY "clients_own_data" ON clients
  FOR ALL USING (org_id = current_setting('app.client_org_id', true));

CREATE POLICY "clients_own_configs" ON module_configs
  FOR ALL USING (client_org_id = current_setting('app.client_org_id', true));

-- Index for fast per-client lookups
CREATE INDEX module_configs_org_module_idx
  ON module_configs (client_org_id, module)
  WHERE enabled = true;
```

### TypeScript Types

```typescript
// lib/types.ts

export interface ClientRecord {
  id: string;
  org_id: string;
  name: string;
  primary_color: string;
  logo_url: string | null;
  active: boolean;
  plan: 'standard' | 'enterprise' | 'pilot';
}

export interface ModuleConfig {
  id: string;
  client_org_id: string;
  module: string;
  system_prompt: string;
  model: string;
  max_tokens: number;
  temperature: number;
  rag_enabled: boolean;
  rag_namespace: string | null;
  enabled: boolean;
}
```

### Provisioning Script

```typescript
// scripts/provision-client.ts
// Run this to onboard a new enterprise client. No deployments. No DevOps.

import { createClient } from '@supabase/supabase-js';
import type { ModuleConfig, ClientRecord } from '../lib/types';

// Service key bypasses RLS — only used in admin scripts, never in app code
const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!
);

interface ProvisionInput {
  orgId: string;
  name: string;
  primaryColor?: string;
  logoUrl?: string;
  plan?: ClientRecord['plan'];
  modules: Array<{
    module: string;
    systemPrompt: string;
    model?: string;
    ragEnabled?: boolean;
    maxTokens?: number;
    temperature?: number;
  }>;
}

export async function provisionClient(input: ProvisionInput): Promise<void> {
  // 1. Insert client record
  const { error: clientError } = await supabase.from('clients').insert({
    org_id:        input.orgId,
    name:          input.name,
    primary_color: input.primaryColor ?? '#2563eb',
    logo_url:      input.logoUrl ?? null,
    plan:          input.plan ?? 'standard',
  });

  if (clientError) {
    throw new Error(`Failed to create client ${input.orgId}: ${clientError.message}`);
  }

  // 2. Insert module configs
  const moduleRows = input.modules.map(m => ({
    client_org_id: input.orgId,
    module:        m.module,
    system_prompt: m.systemPrompt,
    model:         m.model ?? 'claude-sonnet-4-5',
    rag_enabled:   m.ragEnabled ?? false,
    rag_namespace: m.ragEnabled ? `${input.orgId}:${m.module}` : null,
    max_tokens:    m.maxTokens ?? 1024,
    temperature:   m.temperature ?? 0.2,
  }));

  const { error: modulesError } = await supabase
    .from('module_configs')
    .insert(moduleRows);

  if (modulesError) {
    // Roll back client record if modules fail — keep the DB consistent
    await supabase.from('clients').delete().eq('org_id', input.orgId);
    throw new Error(`Failed to create modules for ${input.orgId}: ${modulesError.message}`);
  }

  console.log(`Provisioned: ${input.orgId} — ${input.modules.length} module(s)`);
}

// --- Example: onboard a new utility client ---
await provisionClient({
  orgId:        'nw-utilities',
  name:         'NW Utilities Inc.',
  primaryColor: '#0f4c81',
  plan:         'enterprise',
  modules: [
    {
      module:       'field-ops',
      systemPrompt: `You are FieldBuddy, an AI field operations assistant for NW Utilities.
You help technicians look up work orders, interpret asset readings, and escalate
safety concerns. You have access to the NW Utilities procedure manual. Always cite
the specific procedure number when referencing a documented process.`,
      ragEnabled: true,
    },
    {
      module:       'safety',
      systemPrompt: `You are SafetyBuddy for NW Utilities. You enforce OSHA and
utility-specific safety protocols. When a technician describes a task, identify
applicable safety requirements and required PPE before they proceed. If a task
involves confined space entry or energized equipment, require supervisor sign-off.`,
      ragEnabled:  true,
      temperature: 0,      // No improvisation on safety procedures
    },
  ],
});
```

### Runtime Config Loading

```typescript
// lib/config.ts
// Called on every API request to load the calling client's config

import { createClient } from '@supabase/supabase-js';
import type { ClientRecord, ModuleConfig } from './types';

// Anon key for app requests — RLS enforces isolation
const supabase = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!
);

/**
 * Set the RLS context for this request.
 * Must be called before any DB query that touches client-scoped tables.
 * Without this, RLS returns zero rows (safe default) rather than the wrong rows.
 */
export async function setRlsContext(orgId: string): Promise<void> {
  await supabase.rpc('set_config', {
    setting: 'app.client_org_id',
    value:   orgId,
    is_local: true,       // local = scoped to this transaction only
  });
}

export async function getClientConfig(orgId: string): Promise<ClientRecord> {
  await setRlsContext(orgId);

  const { data, error } = await supabase
    .from('clients')
    .select('*')
    .eq('org_id', orgId)
    .eq('active', true)
    .single();

  if (error || !data) {
    throw new Error(`Unknown or inactive client: ${orgId}`);
  }

  return data;
}

export async function getModuleConfig(
  orgId: string,
  module: string
): Promise<ModuleConfig> {
  await setRlsContext(orgId);

  const { data, error } = await supabase
    .from('module_configs')
    .select('*')
    .eq('client_org_id', orgId)
    .eq('module', module)
    .eq('enabled', true)
    .single();

  if (error || !data) {
    throw new Error(`Module '${module}' not enabled for client '${orgId}'`);
  }

  return data;
}
```

### Dynamic Branding (Client-Side)

```typescript
// lib/branding.ts
// Apply client branding to the shell before rendering any content

import type { ClientRecord } from './types';

export function applyClientBranding(config: ClientRecord): void {
  const root = document.documentElement;

  // CSS custom properties — the entire design system inherits from these
  root.style.setProperty('--color-accent',   config.primary_color);
  root.style.setProperty('--color-accent-10', `${config.primary_color}1a`); // 10% alpha

  if (config.logo_url) {
    const logos = document.querySelectorAll<HTMLImageElement>('[data-client-logo]');
    logos.forEach(img => {
      img.src = config.logo_url!;
      img.alt = config.name;
    });
  }

  // Set page title to client name
  document.title = `${config.name} — BuddyOS`;
}
```

### Tenant Identification at the Edge

```typescript
// middleware.ts (Next.js Edge Middleware)
// Resolve org_id from subdomain before the request hits any handler

import { NextRequest, NextResponse } from 'next/server';

export function middleware(req: NextRequest) {
  const hostname = req.headers.get('host') ?? '';

  // acme.yourplatform.com → org_id = 'acme'
  const subdomain = hostname.split('.')[0];
  const knownSubdomains = ['www', 'app', 'api', 'localhost'];

  if (subdomain && !knownSubdomains.includes(subdomain)) {
    const res = NextResponse.next();
    res.headers.set('x-client-org-id', subdomain);
    return res;
  }

  // Fall back to header if client passes it directly (API consumers)
  const orgIdHeader = req.headers.get('x-client-org-id');
  if (orgIdHeader) return NextResponse.next();

  // No tenant identified — return 401 rather than serving default data
  return new Response(JSON.stringify({ error: 'Tenant not identified' }), {
    status: 401,
    headers: { 'Content-Type': 'application/json' },
  });
}
```

---

## Why It Works

- **Onboarding is a script, not a sprint.** The provisioning script inserts a client and their module configs in under a minute. There are no infrastructure steps. No new subdomains to provision in DNS (if using path-based routing). No new API keys to rotate. The client is live the moment the script completes.
- **RLS is the isolation layer, not application logic.** Application-layer tenant filtering (`WHERE client_org_id = :orgId`) is defense-in-depth, but it can be forgotten. A missing `WHERE` clause in application code is a bug. A misconfigured RLS policy fails closed — you get zero rows, not cross-tenant data. See `cross-tenant-isolation.md` for how to prove this in CI.
- **Config changes don't require deploys.** Updating a client's system prompt, enabling a new module, or adjusting token limits is a database update that takes effect on the next request. No redeployment, no downtime window, no coordination with the platform team.
- **One bug fix, all clients.** When you fix a streaming issue or a citation rendering bug, it ships to all clients in one deploy. The alternative — N separate deployments — means partial fixes in production while you chase the remaining N-1 instances.

---

## Gotchas

**1. `is_local: true` matters in `set_config`.**
Setting `app.client_org_id` without `is_local: true` can leak the value across requests in a connection-pooled environment. Supabase uses PgBouncer in transaction mode by default. If `is_local` is false, the setting persists for the connection's lifetime — which means Request A's tenant context can bleed into Request B's query if they share a pooled connection. Always use `is_local: true`. This is the single most dangerous footgun in Postgres RLS with connection pooling.

**2. Service key access must never reach application request paths.**
The service role bypasses RLS by design — that's why it exists for the provisioning script. If you accidentally wire the service key into an application route (a common copy-paste mistake when prototyping), RLS isolation disappears silently. No error. Just every tenant seeing every other tenant's data. Grep your API route directory for `SERVICE_KEY` before every major ship. It should appear in zero application files.

**3. The `UNIQUE(client_org_id, module)` constraint is load-bearing.**
Without this constraint, running the provisioning script twice on the same client creates duplicate module configs. The runtime config loader uses `.single()` which throws on multiple rows. The constraint converts a data bug into a hard insertion error at provisioning time, which is where you want to catch it. Do not remove it to simplify upsert logic — use `upsert` with `onConflict` instead.

---

## Real-World Note

This pattern is the foundation of the BuddyOS platform. The same codebase currently serves NW Natural (FieldBuddy + InventoryBuddy pilot), multiple standalone product users (GarageBuddy, GuitarBuddy, PoolAndSpaBuddy), and internal development tenants — all from a single Vercel deployment backed by one Supabase project. Adding a new enterprise client takes one provisioning script run and a DNS CNAME if they want a custom subdomain. The entire onboarding takes under 30 minutes including the client call.
