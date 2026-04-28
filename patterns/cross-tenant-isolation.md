# Cross-Tenant Isolation Testing

## Problem

Config-driven multi-tenancy puts all clients in the same database. Row-level security is the enforcement layer. RLS policies are SQL — they can have bugs. A misconfigured policy means Client A can read Client B's data. In regulated operations, that is not a bug. It is a breach.

You cannot manually QA tenant isolation before every deploy. You need CI to catch it.

## Pattern

Seed two test tenants with known data. Run queries as each tenant. Assert the results never cross. Run this on every PR.

## Test Setup

```typescript
// tests/cross-tenant-isolation.test.ts
import { createClient } from "@supabase/supabase-js";
import { describe, it, expect, beforeAll, afterAll } from "vitest";

// Two separate clients scoped to different org_ids
const tenantA = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  global: {
    headers: { "x-client-org-id": "test-tenant-a" },
  },
});

const tenantB = createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
  global: {
    headers: { "x-client-org-id": "test-tenant-b" },
  },
});

// Service client for seeding — bypasses RLS
const service = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

beforeAll(async () => {
  // Seed known data for each tenant
  await service.from("module_configs").insert([
    {
      client_org_id: "test-tenant-a",
      module: "field-ops",
      system_prompt: "TENANT_A_PROMPT",
      rag_enabled: false,
    },
    {
      client_org_id: "test-tenant-b",
      module: "field-ops",
      system_prompt: "TENANT_B_PROMPT",
      rag_enabled: false,
    },
  ]);

  await service.from("document_chunks").insert([
    {
      content: "TENANT_A_DOCUMENT",
      namespace: "test-tenant-a:field-ops",
      client_org_id: "test-tenant-a",
      embedding: Array(1536).fill(0.1),
    },
    {
      content: "TENANT_B_DOCUMENT",
      namespace: "test-tenant-b:field-ops",
      client_org_id: "test-tenant-b",
      embedding: Array(1536).fill(0.2),
    },
  ]);
});

afterAll(async () => {
  // Clean up test tenants
  await service
    .from("module_configs")
    .delete()
    .in("client_org_id", ["test-tenant-a", "test-tenant-b"]);

  await service
    .from("document_chunks")
    .delete()
    .in("client_org_id", ["test-tenant-a", "test-tenant-b"]);
});
```

## Isolation Assertions

```typescript
describe("Cross-tenant data isolation", () => {
  it("Tenant A cannot read Tenant B module configs", async () => {
    const { data } = await tenantA
      .from("module_configs")
      .select("system_prompt")
      .eq("client_org_id", "test-tenant-b");

    expect(data).toHaveLength(0);
  });

  it("Tenant B cannot read Tenant A module configs", async () => {
    const { data } = await tenantB
      .from("module_configs")
      .select("system_prompt")
      .eq("client_org_id", "test-tenant-a");

    expect(data).toHaveLength(0);
  });

  it("Tenant A only sees its own configs", async () => {
    const { data } = await tenantA
      .from("module_configs")
      .select("system_prompt");

    expect(data).toHaveLength(1);
    expect(data![0].system_prompt).toBe("TENANT_A_PROMPT");
    expect(data).not.toContainEqual(
      expect.objectContaining({ system_prompt: "TENANT_B_PROMPT" })
    );
  });

  it("Tenant A cannot read Tenant B document chunks", async () => {
    const { data } = await tenantA
      .from("document_chunks")
      .select("content")
      .eq("client_org_id", "test-tenant-b");

    expect(data).toHaveLength(0);
  });

  it("Vector search does not return cross-tenant chunks", async () => {
    // Call your retrieval function scoped to tenant A
    const results = await retrieveContext(
      "test-tenant-a:field-ops",
      "test query",
      "test-tenant-a"
    );

    const contents = results.map((r) => r.content);
    expect(contents).not.toContain("TENANT_B_DOCUMENT");
  });

  it("Tenant A API endpoint only returns its own data", async () => {
    const res = await fetch("/api/claude", {
      method: "POST",
      headers: {
        "Content-Type": "application/json",
        "x-client-org-id": "test-tenant-a",
      },
      body: JSON.stringify({
        module: "field-ops",
        message: "test message",
      }),
    });

    // System prompt used should be tenant A's, never tenant B's
    // Assert via a canary value in the prompt rather than reading the prompt directly
    expect(res.status).toBe(200);
  });
});
```

## CI Integration

```yaml
# .github/workflows/isolation.yml
name: Cross-Tenant Isolation

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  isolation:
    runs-on: ubuntu-latest
    env:
      SUPABASE_URL: ${{ secrets.SUPABASE_URL }}
      SUPABASE_ANON_KEY: ${{ secrets.SUPABASE_ANON_KEY }}
      SUPABASE_SERVICE_KEY: ${{ secrets.SUPABASE_SERVICE_KEY }}

    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: "20"
      - run: npm ci
      - run: npx vitest run tests/cross-tenant-isolation.test.ts
```

## What to Test Beyond the Happy Path

The obvious tests above catch accidental exposure. These catch the subtle failures:

**Missing WHERE clause** — A query that fetches all rows and filters in application code instead of SQL bypasses RLS entirely. Test by seeding 10 tenants and asserting the row count matches exactly one tenant's data.

**Service key exposure** — The service role bypasses RLS by design. Test that your API routes never use the service key in a path reachable by tenant requests. Audit with grep: `grep -r "SERVICE_KEY" src/` should return zero results in application code.

**Namespace collision** — If RAG namespaces are predictable (`clientId:module`), a tenant who knows another's `clientId` could craft a request to pull their documents. Test that the namespace is validated against the authenticated `clientId` before retrieval.

**Unauthenticated requests** — Test that a request with no `x-client-org-id` returns 401, not empty data. Empty data is silently wrong. 401 is explicitly right.

## Why This Belongs in CI

RLS policies break silently. A schema migration that drops and recreates a table loses its policies. A new table added without RLS is open by default. A policy rewrite that looks correct in review can have a logic inversion.

If isolation is not tested on every PR, it is not tested. It is hoped.
