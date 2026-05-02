# Pattern: Cross-Tenant Isolation Testing

## Problem

Config-driven multi-tenancy puts all clients in the same database. Row-level security is the enforcement layer. RLS policies are SQL — they can have bugs. A schema migration that drops and recreates a table silently loses its RLS policies. A new table added without RLS is open by default. A policy rewrite that looks correct in code review has a logic inversion that nobody catches until a client reports seeing data they shouldn't.

In regulated industries — utilities, construction, healthcare operations — cross-tenant data exposure is not a software defect. It is a breach with contractual, legal, and reputational consequences. You cannot manually QA tenant isolation before every deploy. You need CI to catch it automatically on every PR that touches the database layer.

This pattern is the "prove it in CI" approach. Seed two test tenants with known data. Run queries as each tenant. Assert zero cross-contamination. Make it a required check before merge. When an auditor asks how you enforce tenant isolation, you show them a green CI run, not a whiteboard diagram.

---

## Solution

### Test Infrastructure

```typescript
// tests/cross-tenant-isolation.test.ts

import { createClient, SupabaseClient } from '@supabase/supabase-js';
import { describe, it, expect, beforeAll, afterAll } from 'vitest';
import { retrieveContext } from '../lib/rag-citations';

// Validate required env vars before the test suite runs
const SUPABASE_URL       = process.env.SUPABASE_URL!;
const SUPABASE_ANON_KEY  = process.env.SUPABASE_ANON_KEY!;
const SUPABASE_SERVICE_KEY = process.env.SUPABASE_SERVICE_KEY!;

if (!SUPABASE_URL || !SUPABASE_ANON_KEY || !SUPABASE_SERVICE_KEY) {
  throw new Error('Missing required Supabase env vars for isolation tests');
}

// Two Supabase clients scoped to different org_ids.
// In the real app, org_id comes from middleware (subdomain or header).
// In tests, we inject it directly in the client header.
function makeTenantClient(orgId: string): SupabaseClient {
  return createClient(SUPABASE_URL, SUPABASE_ANON_KEY, {
    global: {
      headers: { 'x-client-org-id': orgId },
    },
  });
}

const tenantA = makeTenantClient('test-tenant-a');
const tenantB = makeTenantClient('test-tenant-b');

// Service client bypasses RLS — used only for seeding and teardown.
// NEVER used in application code paths.
const service = createClient(SUPABASE_URL, SUPABASE_SERVICE_KEY);

// Known sentinel values — if these appear in the wrong tenant's query results,
// the test fails with a clear signal, not a vague data mismatch
const SENTINEL_A_PROMPT   = 'ISOLATION_TEST__TENANT_A_SYSTEM_PROMPT';
const SENTINEL_B_PROMPT   = 'ISOLATION_TEST__TENANT_B_SYSTEM_PROMPT';
const SENTINEL_A_DOCUMENT = 'ISOLATION_TEST__TENANT_A_DOCUMENT_CONTENT';
const SENTINEL_B_DOCUMENT = 'ISOLATION_TEST__TENANT_B_DOCUMENT_CONTENT';

beforeAll(async () => {
  // Seed clients
  await service.from('clients').upsert([
    {
      org_id:        'test-tenant-a',
      name:          'Isolation Test Tenant A',
      primary_color: '#ff0000',
      active:        true,
    },
    {
      org_id:        'test-tenant-b',
      name:          'Isolation Test Tenant B',
      primary_color: '#0000ff',
      active:        true,
    },
  ], { onConflict: 'org_id' });

  // Seed module configs
  await service.from('module_configs').upsert([
    {
      client_org_id: 'test-tenant-a',
      module:        'field-ops',
      system_prompt: SENTINEL_A_PROMPT,
      rag_enabled:   false,
      enabled:       true,
    },
    {
      client_org_id: 'test-tenant-b',
      module:        'field-ops',
      system_prompt: SENTINEL_B_PROMPT,
      rag_enabled:   false,
      enabled:       true,
    },
  ], { onConflict: 'client_org_id,module' });

  // Seed document chunks — use embeddings that are clearly different
  // so vector retrieval results are deterministic in tests
  await service.from('document_chunks').upsert([
    {
      content:       SENTINEL_A_DOCUMENT,
      namespace:     'test-tenant-a:field-ops',
      client_org_id: 'test-tenant-a',
      source:        'test-doc-a.pdf',
      page:          1,
      // All-0.1 embedding — low norm, but enough for the similarity filter
      embedding:     Array(1536).fill(0.1),
    },
    {
      content:       SENTINEL_B_DOCUMENT,
      namespace:     'test-tenant-b:field-ops',
      client_org_id: 'test-tenant-b',
      source:        'test-doc-b.pdf',
      page:          1,
      embedding:     Array(1536).fill(0.2),
    },
  ], { onConflict: 'id' });
});

afterAll(async () => {
  // Clean up test data — use service key to bypass RLS for teardown
  await service
    .from('document_chunks')
    .delete()
    .in('client_org_id', ['test-tenant-a', 'test-tenant-b']);

  await service
    .from('module_configs')
    .delete()
    .in('client_org_id', ['test-tenant-a', 'test-tenant-b']);

  await service
    .from('clients')
    .delete()
    .in('org_id', ['test-tenant-a', 'test-tenant-b']);
});
```

### Isolation Assertions

```typescript
describe('Cross-tenant data isolation — module_configs', () => {

  it('Tenant A cannot read Tenant B module configs by explicit filter', async () => {
    const { data } = await tenantA
      .from('module_configs')
      .select('system_prompt')
      .eq('client_org_id', 'test-tenant-b');

    // RLS should return zero rows, not an error.
    // An error here means the policy is misconfigured — not that isolation works.
    expect(data).toHaveLength(0);
  });

  it('Tenant B cannot read Tenant A module configs by explicit filter', async () => {
    const { data } = await tenantB
      .from('module_configs')
      .select('system_prompt')
      .eq('client_org_id', 'test-tenant-a');

    expect(data).toHaveLength(0);
  });

  it('Tenant A full-table SELECT returns only its own configs', async () => {
    const { data } = await tenantA
      .from('module_configs')
      .select('system_prompt');

    // Exactly one row — its own. Not zero (that would mean RLS is blocking
    // the tenant from its own data), not two (that would mean cross-leak).
    expect(data).toHaveLength(1);
    expect(data![0].system_prompt).toBe(SENTINEL_A_PROMPT);
  });

  it('Tenant A full-table SELECT never includes Tenant B sentinel', async () => {
    const { data } = await tenantA
      .from('module_configs')
      .select('system_prompt');

    const prompts = (data ?? []).map(r => r.system_prompt);
    expect(prompts).not.toContain(SENTINEL_B_PROMPT);
  });

  it('Tenant B full-table SELECT returns only its own configs', async () => {
    const { data } = await tenantB
      .from('module_configs')
      .select('system_prompt');

    expect(data).toHaveLength(1);
    expect(data![0].system_prompt).toBe(SENTINEL_B_PROMPT);
  });
});

describe('Cross-tenant data isolation — document_chunks', () => {

  it('Tenant A cannot read Tenant B documents by explicit filter', async () => {
    const { data } = await tenantA
      .from('document_chunks')
      .select('content')
      .eq('client_org_id', 'test-tenant-b');

    expect(data).toHaveLength(0);
  });

  it('Tenant A full-table SELECT never includes Tenant B document', async () => {
    const { data } = await tenantA
      .from('document_chunks')
      .select('content');

    const contents = (data ?? []).map(r => r.content);
    expect(contents).not.toContain(SENTINEL_B_DOCUMENT);
  });

  it('Vector retrieval for Tenant A never returns Tenant B chunks', async () => {
    // This tests the pgvector RPC function, not just the table SELECT.
    // The SQL function must also respect the RLS context.
    const results = await retrieveContext(
      'test-tenant-a:field-ops',
      'test isolation query',
      'test-tenant-a',
      10   // High topK to maximize chance of catching a leak
    );

    const contents = results.map(r => r.content);
    expect(contents).not.toContain(SENTINEL_B_DOCUMENT);
  });

  it('Vector retrieval for Tenant B never returns Tenant A chunks', async () => {
    const results = await retrieveContext(
      'test-tenant-b:field-ops',
      'test isolation query',
      'test-tenant-b',
      10
    );

    const contents = results.map(r => r.content);
    expect(contents).not.toContain(SENTINEL_A_DOCUMENT);
  });
});

describe('Cross-tenant isolation — edge cases', () => {

  it('Request with no org_id header is rejected at the API layer', async () => {
    // Unauthenticated request — no x-client-org-id header
    const res = await fetch(`${process.env.NEXT_PUBLIC_BASE_URL}/api/chat`, {
      method:  'POST',
      headers: { 'Content-Type': 'application/json' },
      body:    JSON.stringify({ module: 'field-ops', message: 'test' }),
    });

    // Must return 401, not 200 with empty data.
    // Empty data is silently wrong. 401 is explicitly right.
    expect(res.status).toBe(401);
  });

  it('Multi-row seed produces exactly N rows per tenant — not N×2', async () => {
    // Guards against a broken UNIQUE constraint allowing duplicate configs
    const { data: aData } = await tenantA
      .from('module_configs')
      .select('id')
      .eq('module', 'field-ops');

    const { data: bData } = await tenantB
      .from('module_configs')
      .select('id')
      .eq('module', 'field-ops');

    expect(aData).toHaveLength(1);
    expect(bData).toHaveLength(1);
  });

  it('Tenant A client record is not visible to Tenant B', async () => {
    const { data } = await tenantB
      .from('clients')
      .select('org_id, name')
      .eq('org_id', 'test-tenant-a');

    expect(data).toHaveLength(0);
  });
});
```

### CI Integration

```yaml
# .github/workflows/isolation.yml

name: Cross-Tenant Isolation

on:
  pull_request:
    branches: [main]
    paths:
      # Run on any change that could affect data isolation
      - 'supabase/migrations/**'
      - 'lib/config.ts'
      - 'lib/rag-citations.ts'
      - 'app/api/**'
      - 'tests/cross-tenant-isolation.test.ts'
  push:
    branches: [main]

jobs:
  isolation:
    name: Verify tenant isolation
    runs-on: ubuntu-latest
    environment: test     # Separate GH environment with test Supabase credentials

    env:
      SUPABASE_URL:          ${{ secrets.TEST_SUPABASE_URL }}
      SUPABASE_ANON_KEY:     ${{ secrets.TEST_SUPABASE_ANON_KEY }}
      SUPABASE_SERVICE_KEY:  ${{ secrets.TEST_SUPABASE_SERVICE_KEY }}
      NEXT_PUBLIC_BASE_URL:  ${{ secrets.TEST_APP_BASE_URL }}

    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Run isolation tests
        run: npx vitest run tests/cross-tenant-isolation.test.ts --reporter=verbose

      - name: Upload test results
        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: isolation-test-results
          path: test-results/
          retention-days: 30    # Keep audit trail for 30 days
```

---

## Why It Works

- **Sentinel values produce unambiguous failures.** Using strings like `ISOLATION_TEST__TENANT_A_SYSTEM_PROMPT` instead of generic test data means that if the wrong value appears in a query result, the assertion failure tells you exactly which tenant's data leaked and into which query. No guesswork.
- **Testing the RPC function is as important as testing the table.** RLS on `document_chunks` protects direct table access. But your pgvector retrieval function is a SQL function that runs with the calling user's permissions — if that function doesn't honor RLS (or is defined with `SECURITY DEFINER`), the table policy is irrelevant. Testing through `retrieveContext` covers the full retrieval stack.
- **The unauthenticated 401 test is not redundant.** A system that returns zero rows for an unauthenticated request looks correct in query results but is silently dangerous — it means unauthenticated requests are reaching the database at all. The 401 test enforces that the middleware gate is working, not just that RLS happens to produce empty results for unknown tenants.
- **`paths:` filter in CI keeps the check fast.** Isolation tests only run when files that could break isolation change. This keeps the CI feedback loop tight without skipping the check entirely. Add new files to the `paths:` list when you add new data access layers.

---

## Gotchas

**1. RLS policies are silently dropped on table recreation.**
`DROP TABLE` + `CREATE TABLE` in a migration loses RLS policies. `ALTER TABLE` + `TRUNCATE` do not. Always verify that your migration scripts use `ALTER TABLE` for schema changes, not drop-recreate. Add a test that directly queries `pg_policies` to assert that RLS is enabled on every table that holds client data — this is the canary check for migration accidents.

```typescript
it('RLS is enabled on all client-scoped tables', async () => {
  const { data } = await service.rpc('check_rls_enabled', {
    table_names: ['clients', 'module_configs', 'document_chunks'],
  });
  data!.forEach(row => expect(row.rls_enabled).toBe(true));
});
```

**2. Connection pooling breaks per-transaction `set_config` if you misconfigure it.**
`set_config('app.client_org_id', value, is_local := true)` scopes the setting to the current transaction. In Supabase's PgBouncer transaction-mode pooling, that transaction ends when your query completes. This is correct behavior — but if you're running multiple queries in a single request without resetting the config between them, and a previous request's connection is reused, there's a brief window where the config is unset. Always call `setRlsContext` at the start of every handler that touches client-scoped tables, not once at middleware level.

**3. These tests must run against a real database, not mocks.**
The entire point of these tests is to verify that the database's RLS policies work as expected. Mocking Supabase defeats the test. Use a dedicated test Supabase project (separate from staging and production) with its own credentials in a separate GitHub Actions environment. The test project should run your full migration history, including any RLS policy changes, so the schema matches production exactly.

---

## Real-World Note

This test suite was written after a schema migration that added a `client_sessions` table without enabling RLS. The table was only used for analytics and contained no sensitive data — but it was visible to all tenants, which would have been a disclosure issue under the enterprise contracts. The isolation tests caught the missing policy in the PR, before it reached staging. The fix was two lines of SQL. Without this test, it would have shipped to production and required a client notification. With it, it was a normal code review comment.
