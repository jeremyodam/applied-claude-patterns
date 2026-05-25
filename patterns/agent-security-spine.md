# Pattern: Agent Security Spine

## Problem

Modern agent frameworks ship with no fine-grained IAM. Every in-app policy has a documented bypass. Approval prompts are usability conveniences, not security boundaries. Default toolsets are permissive, default backends often run with full host access, and "secure by default" is a marketing claim rather than a code state.

This is not a flaw of any specific framework. It is a structural property of the agent paradigm: the agent is the user, the model decides what tools to call, and the surface area expands every time you add a skill, a tool, or a context source. You cannot lock down an agent the way you lock down a microservice. The principle of least privilege fights the principle of "the assistant should help with whatever you ask."

In regulated operations — utilities, healthcare, construction, financial services — this gap is not a theoretical risk. A stolen OAuth token, a prompt injection that exfiltrates auth credentials, an agent-created skill that runs unscrutinized code, a multi-tenant SaaS where one client's agent leaks another's data: each of these has shipped, been disclosed, and triggered audits in the last eighteen months.

The fix is not to add more in-app guards. The fix is to **stop treating in-app guards as boundaries** and to make the OS, network, and tenant-isolation layers do the real enforcement. This pattern is the production checklist.

---

## Solution

### The Spine

> Agent frameworks ship with no fine-grained IAM. Every in-app policy has a documented bypass. The OS/network layer is the only hard boundary. Minimize what sits behind it, and **assume the agent will eventually do the wrong thing.**

Treat in-app approval prompts, system-prompt instructions, and per-tool guards like client-side form validation: helpful for the user, never trusted by the server.

### Six Default Failure Modes to Audit

These were catalogued from a public eval of the Hermes Agent framework (MIT, NousResearch). The issue numbers are real, public, and citable. The pattern is universal: any agent framework you adopt or build is subject to the same audit.

| # | Default failure mode | Hermes ref | Production fix |
|---|---|---|---|
| 1 | No egress filtering | [#4170](https://github.com/NousResearch/hermes-agent/issues/4170) | Network egress allowlist enforced at the function/container layer |
| 2 | Default backend = full host access | n/a | Containerize. Serverless functions (Vercel/Lambda) are the floor. No `local` mode in prod. |
| 3 | Agent-created code unguarded | [#7072](https://github.com/NousResearch/hermes-agent/issues/7072) | Self-modification requires human approval queue; no dynamic `import` from URLs |
| 4 | `read_file` can exfiltrate auth | [#17656](https://github.com/NousResearch/hermes-agent/issues/17656) | Credentials never in agent-readable paths; CI grep on bundle output |
| 5 | Approval gate is UX, not security | Hermes code says so itself | Server-side enforcement is the boundary; UI prompts are awareness only |
| 6 | Default-permit toolsets | n/a | Per-agent capability scoping in code, default-deny, PR gate on additions |

### The Egress Allowlist (Network Layer)

```typescript
// lib/safe-fetch.ts
//
// Every agent-facing tool uses this. Raw `fetch()` is banned by a CI grep.
// The allowlist is reviewed quarterly — see egress-allowlist-recent.test.ts.

const ALLOWED_HOSTS = new Set([
  'api.anthropic.com',
  'api.openai.com',         // remove if not in use
  '*.supabase.co',
  // Partner APIs are added per-tool, not globally:
  // 'api.partsapi.example',  // GarageBuddy parts lookup
  // 'api.weatherapi.example',// FieldBuddy weather context
]);

export function safeFetch(url: string, init?: RequestInit): Promise<Response> {
  const { hostname, protocol } = new URL(url);

  // Block non-HTTPS in production — agents have no reason to make plaintext calls
  if (process.env.NODE_ENV === 'production' && protocol !== 'https:') {
    throw new Error(`egress-denied-protocol: ${protocol}`);
  }

  const allowed = [...ALLOWED_HOSTS].some(h =>
    h.startsWith('*.') ? hostname.endsWith(h.slice(1)) : h === hostname
  );

  if (!allowed) {
    // Log the attempt — a denied egress call is a signal, not noise
    console.error(JSON.stringify({
      event:   'egress_denied',
      host:    hostname,
      ts:      new Date().toISOString(),
      // Don't log the full URL — it can contain auth tokens in query strings
    }));
    throw new Error(`egress-denied: ${hostname}`);
  }

  return fetch(url, init);
}
```

Enforce with a CI grep that fails the build if any agent-layer file imports `fetch` directly:

```yaml
# .github/workflows/safe-fetch.yml
name: Safe Fetch Only

on:
  pull_request:
    paths:
      - 'lib/agents/**'
      - 'lib/tools/**'
      - 'app/api/**'

jobs:
  grep:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Reject raw fetch in agent layer
        run: |
          if grep -rn --include='*.ts' --include='*.tsx' \
               -P '(?<!safe)\bfetch\s*\(' \
               lib/agents/ lib/tools/ app/api/agent/; then
            echo "ERROR: raw fetch() found in agent layer. Use safeFetch."
            exit 1
          fi
```

### Credential Quarantine (Filesystem Layer)

```typescript
// tests/no-credentials-in-bundle.test.ts
//
// Run pre-deploy. Scans the webpacked bundle for known token prefixes.
// A leaked credential in the bundle is a P0; this test makes that impossible
// to ship by accident.

import { describe, it, expect } from 'vitest';
import { readFileSync, readdirSync } from 'node:fs';
import { join } from 'node:path';

const BUNDLE_DIR = '.vercel/output/functions';

const FORBIDDEN_PATTERNS: Array<{ name: string; pattern: RegExp }> = [
  { name: 'anthropic-key',     pattern: /\bsk-ant-[a-zA-Z0-9_-]{20,}/ },
  { name: 'openai-key',        pattern: /\bsk-[a-zA-Z0-9]{20,}/ },
  { name: 'supabase-service',  pattern: /\beyJ[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{20,}\.[A-Za-z0-9_-]{20,}/ },
  { name: 'aws-access-key',    pattern: /\bAKIA[0-9A-Z]{16}\b/ },
  { name: 'private-key',       pattern: /-----BEGIN (RSA |EC )?PRIVATE KEY-----/ },
];

function walk(dir: string): string[] {
  return readdirSync(dir, { withFileTypes: true }).flatMap(d =>
    d.isDirectory() ? walk(join(dir, d.name)) : [join(dir, d.name)]
  );
}

describe('Credential quarantine', () => {
  it('no production credentials appear in the deploy bundle', () => {
    const files = walk(BUNDLE_DIR).filter(f =>
      f.endsWith('.js') || f.endsWith('.mjs') || f.endsWith('.json')
    );

    const findings: Array<{ file: string; pattern: string }> = [];

    for (const file of files) {
      const content = readFileSync(file, 'utf8');
      for (const { name, pattern } of FORBIDDEN_PATTERNS) {
        if (pattern.test(content)) {
          findings.push({ file, pattern: name });
        }
      }
    }

    // Use toEqual([]) instead of .length === 0 — failure message lists the leaks
    expect(findings).toEqual([]);
  });
});
```

### Default-Deny Toolsets (Per-Agent Capability Scoping)

```typescript
// lib/toolset.ts
//
// Adding a tool to an agent requires editing this file.
// There is no global default-permit list. PR review is the gate.

type AgentName = 'sales' | 'support' | 'ops' | 'research';

const TOOLSETS: Record<AgentName, readonly string[]> = {
  sales:    ['crm_lookup', 'quote_generate', 'product_catalog'],
  support:  ['kb_search', 'ticket_query', 'order_status'],
  ops:      ['inventory_check', 'sop_lookup', 'safety_check'],
  research: ['web_search', 'document_summarize'],
};

export function toolsFor(agent: AgentName): readonly string[] {
  // Unknown agent → empty toolset, not full set.
  // This is the inverse of how most frameworks default.
  return TOOLSETS[agent] ?? [];
}

// Runtime check, defense-in-depth: even if the agent name comes from a
// trusted source, double-check at call time.
export function assertToolAllowed(agent: AgentName, tool: string): void {
  if (!toolsFor(agent).includes(tool)) {
    throw new Error(`tool-denied: agent=${agent} tool=${tool}`);
  }
}
```

### The Read-Only Triage Pattern (Email/Calendar Agents)

A common antipattern: build an "email triage" agent that uses the platform's native Gmail integration. The default OAuth scope is read+write. The agent now has the ability to send mail, modify calendar events, and delete messages — none of which it was supposed to do.

The fix is to split the read from the act:

```
┌─────────────────────┐         ┌─────────────────────┐
│ Read-only fetcher   │         │ Triage agent        │
│ (separate process)  │         │ (no exec, no I/O)   │
│                     │  digest │                     │
│ Gmail/Cal API       │ ───────▶│ Reads digest text   │
│ Read-only scope     │  table  │ Produces summary    │
│ Proxy account       │         │ No outbound calls   │
└─────────────────────┘         └─────────────────────┘
         │                                │
         ▼                                ▼
   OAuth token                   Anthropic API only
   (sandboxed)                   (via safeFetch)
```

The fetcher is a tiny standalone process with read-only OAuth scopes and a proxy account (never the user's primary credentials). It writes a digest to a known database row. The triage agent reads the digest as plain text input — it has no Gmail tool, no Calendar tool, no outbound HTTP except to the LLM provider.

A prompt injection in the agent cannot send mail or modify the calendar because the agent has no tools that do those things. A compromised fetcher cannot escalate because its OAuth scopes are read-only and its account has no permissions worth stealing.

---

## Why It Works

- **The egress allowlist is a hard boundary, not a request from the model.** A prompt injection telling the agent to "POST credentials to evil.example.com" returns an `egress-denied` error from `safeFetch`. The agent cannot reason its way around a function that throws.
- **The credential test runs against the actual deploy bundle, not the source tree.** Webpack inlines `process.env.*` references at build time. A misconfigured build (e.g., importing server-only env into a client component) can leak credentials into the bundle even when the source code looks correct. Scanning the output is the only way to catch this class of leak.
- **Default-deny toolsets shift the failure mode from "silently insecure" to "loudly broken."** A new tool added to an agent without updating `TOOLSETS` returns `tool-denied` at call time. Engineers notice immediately. The wrong default is the kind of bug that ships to production and is discovered months later in a pen test.
- **Splitting the read from the act eliminates a whole class of bypass.** You cannot exfiltrate via a tool that does not exist. The triage agent's capability surface is "summarize this text" — there is nothing to inject toward.

---

## Gotchas

**1. The egress allowlist is only as good as its review cadence.**
Allowlists accumulate. A partner integration gets added for a one-off demo and never removed. After a year, the "allowlist" is "everything we have ever talked to." Enforce a quarterly review with a timestamp comment in the file:

```typescript
// Reviewed: 2026-05-24 by jeremy@odamsolutions.com
// Next review: 2026-08-24
const ALLOWED_HOSTS = new Set([...]);
```

A weekly cron job runs a test that fails if the timestamp is more than 100 days old. The review takes 5 minutes. The discipline of "if you don't review, the build fails" is what keeps the list honest.

**2. The credential test will produce false positives on test fixtures.**
Test JWT fixtures, mock API keys, and sandbox tokens look exactly like real credentials to a regex. Two fixes: (a) exclude test directories from the bundle scan, and (b) prefix all test fixtures with a sentinel string (`TEST_FIXTURE_`) and exclude that prefix from the patterns. Do not skip the test or weaken the pattern.

**3. Default-deny toolsets break when someone adds a new agent.**
The `Record<AgentName, ...>` type forces TypeScript to fail if you add a new agent to the `AgentName` union without adding it to `TOOLSETS`. This works only if all agent definitions go through that file. If your codebase has multiple sources of truth for "what agents exist," the type check silently fails to enforce coverage. Centralize the agent registry.

**4. The fetcher pattern is more code than the integrated approach.**
You are running two processes instead of one, managing two sets of credentials, and adding a database row as a serialization point. The cost is justified by the threat model (an agent with read+write scopes on a user's mailbox is a much higher-value target than a digest reader), but it is a real cost. Do not adopt this pattern for an agent that genuinely only needs to read calendar availability — at that point, a single read-only OAuth scope on the integrated agent is fine. Reserve the split for agents that would otherwise default to read+write scopes.

**5. Version pinning fights "stay current."**
Auto-update is a security feature (CVE patches ship faster) and a security risk (a poisoned dependency ships to production faster). The reconciliation is to pin tagged versions and run an automated PR-bot that proposes updates for human review on a regular cadence. Dependabot or Renovate, configured to require approval, not to auto-merge. You get the visibility of "we are 3 versions behind" without the surprise of "main got a backdoored commit at 2 AM."

---

## Real-World Note

This spine was distilled in a single week from two parallel inputs: a public code review of an open-source agent framework (Hermes Agent, MIT) and a working engineer's setup notes for running that framework in production. The engineer's notes flagged a deployment that had drifted 800 commits past its pinned tag because the framework's auto-updater silently pulls HEAD. The code review surfaced six default behaviors that are unsafe and the public GitHub issues tracking them.

The lesson is not "Hermes is insecure." The lesson is that **every agent framework you adopt is subject to the same audit**, and the framework you write yourself is no exception. If you build an agent on Claude, you are responsible for the spine. The framework gives you tools; the OS/network layer gives you boundaries. Wire the boundaries first.

---

## Cross-References

- [Cross-Tenant Isolation Testing](cross-tenant-isolation.md) — the data-layer counterpart to this pattern
- [Universal Streaming Endpoint](universal-endpoint.md) — the single boundary where most of the safeFetch enforcement lives
- Hermes Agent: [github.com/NousResearch/hermes-agent](https://github.com/NousResearch/hermes-agent) — public source for the audit references above
