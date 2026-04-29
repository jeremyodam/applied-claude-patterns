# Config-Driven Multi-Tenancy

## Problem

You have one AI platform and multiple enterprise clients. Each client needs their own branding, their own system prompt, their own document corpus, and their own set of enabled modules. The naive solution is a separate deployment per client. That means N codebases to maintain, N deploy pipelines, and N places for bugs to live.

The better architecture: one codebase, one deployment, N clients configured entirely through data.

## Pattern

Every client has a row in a `clients` table and rows in `module_configs`. The application reads config at request time and renders accordingly. Onboarding a new client means inserting rows, not deploying code.

## Schema

```sql
-- clients table — one row per enterprise client
CREATE TABLE clients (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  org_id text UNIQUE NOT NULL,       -- slug used in API requests and RLS
  name text NOT NULL,
  primary_color text DEFAULT '#2563eb',
  logo_url text,
  active boolean DEFAULT true,
  created_at timestamptz DEFAULT now()
);

-- module_configs — which AI modules are enabled per client, with what config
CREATE TABLE module_configs (
  id uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  client_org_id text REFERENCES clients(org_id),
  module text NOT NULL,              -- 'field-ops', 'safety', 'fleet', etc.
  system_prompt text NOT NULL,
  model text DEFAULT 'claude-sonnet-4-5',
  max_tokens int DEFAULT 1024,
  rag_enabled boolean DEFAULT false,
  rag_namespace text,               -- scopes vector retrieval to this client+module
  enabled boolean DEFAULT true,
  UNIQUE(client_org_id, module)
);

-- Row-level security — clients only see their own configs
ALTER TABLE module_configs ENABLE ROW LEVEL SECURITY;

CREATE POLICY "clients_own_configs" ON module_configs
  FOR ALL USING (client_org_id = current_setting('app.client_org_id'));
```

## Provisioning Script

```typescript
// scripts/provision-client.ts — run once per new client
import { createClient } from "@supabase/supabase-js";

const supabase = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

interface ClientProvisionConfig {
  orgId: string;
  name: string;
  primaryColor?: string;
  logoUrl?: string;
  modules: ModuleConfig[];
}

interface ModuleConfig {
  module: string;
  systemPrompt: string;
  ragEnabled?: boolean;
  ragNamespace?: string;
  maxTokens?: number;
}

export async function provisionClient(config: ClientProvisionConfig) {
  // 1. Insert client record
  const { error: clientError } = await supabase.from("clients").insert({
    org_id: config.orgId,
    name: config.name,
    primary_color: config.primaryColor ?? "#2563eb",
    logo_url: config.logoUrl,
  });

  if (clientError) throw clientError;

  // 2. Insert module configs
  const moduleRows = config.modules.map((m) => ({
    client_org_id: config.orgId,
    module: m.module,
    system_prompt: m.systemPrompt,
    rag_enabled: m.ragEnabled ?? false,
    rag_namespace: m.ragNamespace ?? `${config.orgId}:${m.module}`,
    max_tokens: m.maxTokens ?? 1024,
  }));

  const { error: modulesError } = await supabase
    .from("module_configs")
    .insert(moduleRows);

  if (modulesError) throw modulesError;

  console.log(`Provisioned client: ${config.orgId} with ${config.modules.length} modules`);
}

// Example: onboard a new client in 10 lines
await provisionClient({
  orgId: "acme-corp",
  name: "ACME Utilities",
  primaryColor: "#dc2626",
  modules: [
    {
      module: "field-ops",
      systemPrompt: "You are a field operations assistant for ACME Utilities...",
      ragEnabled: true,
    },
    {
      module: "safety",
      systemPrompt: "You are a safety compliance assistant grounded in applicable industry safety standards...",
      ragEnabled: true,
      ragNamespace: "acme-corp:safety",
    },
  ],
});
```

## Runtime Config Loading

```typescript
// lib/config.ts — load client config at request time
export async function getClientConfig(orgId: string): Promise<ClientConfig> {
  const { data } = await supabase
    .from("clients")
    .select("*")
    .eq("org_id", orgId)
    .eq("active", true)
    .single();

  if (!data) throw new Error(`Unknown client: ${orgId}`);
  return data;
}

export async function getModuleConfig(
  orgId: string,
  module: string
): Promise<ModuleConfig> {
  // Set RLS context so DB enforces isolation
  await supabase.rpc("set_config", {
    setting: "app.client_org_id",
    value: orgId,
  });

  const { data } = await supabase
    .from("module_configs")
    .select("*")
    .eq("client_org_id", orgId)
    .eq("module", module)
    .eq("enabled", true)
    .single();

  if (!data) throw new Error(`Module ${module} not enabled for ${orgId}`);
  return data;
}
```

## Dynamic Branding

```typescript
// Inject client branding as CSS variables at the app shell level
export function applyClientBranding(config: ClientConfig) {
  document.documentElement.style.setProperty(
    "--accent",
    config.primary_color
  );

  if (config.logo_url) {
    const logo = document.getElementById("client-logo") as HTMLImageElement;
    if (logo) logo.src = config.logo_url;
  }
}
```

## Deployment Pattern

One Vercel project. One Supabase project. Client is identified by:
- Subdomain (`acme.yourplatform.com`) resolved at the edge
- API key header scoped to `org_id` at auth time
- Or path prefix (`/app/acme-corp/`) for simpler setups

All three patterns set `orgId` before any DB call. RLS does the rest.

## Why This Beats Separate Deployments

| | Separate Deployments | Config-Driven |
|---|---|---|
| New client onboarding | Deploy new instance (~1 day) | Insert rows (~10 min) |
| Bug fix | Patch N deployments | Patch once |
| Model upgrade | Update N env vars | Update one default |
| Client isolation | Network-level | RLS + runtime scoping |
| Cost | N × infrastructure | Fixed |

The tradeoff: a misconfigured RLS policy leaks data across tenants. See the [Cross-Tenant Isolation Testing](cross-tenant-isolation.md) pattern for how to catch that before it ships.
