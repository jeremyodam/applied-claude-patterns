# Pattern: Universal Streaming Endpoint

## Problem

When you're running 12+ AI products off one Anthropic account, the instinct is to spin up a separate API route for each one. That's how you end up with `/api/garagebuddy/chat`, `/api/digbuddy/chat`, `/api/askbuddy/chat` — all doing the same thing with slightly different system prompts hardcoded in the handler. Now you have 12 places to update when the SDK bumps a version, 12 places where a developer can accidentally leak the full system prompt in a `console.log`, and 12 independent cold-start paths on Vercel. This pattern collapses all of that into one endpoint driven by a config registry. One deploy. One place to instrument. One place to change model selection when Haiku ships a new version. Adding product #13 is a config block, not a deploy.

---

## Solution

### Product Config Registry

```typescript
// lib/buddy-configs.ts

import Anthropic from '@anthropic-ai/sdk';

export type BuddyModel =
  | 'claude-haiku-4-5'
  | 'claude-sonnet-4-5'
  | 'claude-opus-4-5';

export interface BuddyConfig {
  productId: string;
  model: BuddyModel;
  maxTokens: number;
  systemPrompt: string;
  /**
   * Temperature 0 = deterministic procedures.
   * Temperature 1 = creative / conversational.
   * Pick based on whether a wrong answer is a liability.
   */
  temperature: number;
  stream: boolean;
  tools?: Anthropic.Tool[];
}

export const BUDDY_CONFIGS: Record<string, BuddyConfig> = {
  garagebuddy: {
    productId: 'garagebuddy',
    model: 'claude-sonnet-4-5',
    maxTokens: 1024,
    temperature: 0.2,
    stream: true,
    systemPrompt: `You are GarageBuddy, an automotive diagnostic assistant for field
technicians and car owners. You interpret OBD-II diagnostic trouble codes (DTCs),
explain sensor readings, and recommend repair paths.

When a DTC is present, always explain: (1) what system it affects, (2) likely root
causes ranked by frequency, (3) what to check first with basic tools.

You do NOT recommend parts without a diagnostic path. You do NOT guess. If the data
is ambiguous, say so and ask for more sensor readings. Respond in plain language —
the user may be under a car in a parking lot.`,
  },

  digbuddy: {
    productId: 'digbuddy',
    model: 'claude-haiku-4-5',     // Fast + cheap — most queries are narrow 811 lookups
    maxTokens: 512,
    temperature: 0,                  // Zero tolerance for hallucinated dig law
    stream: true,
    systemPrompt: `You are DigBuddy, a bilingual (EN/ES) 811 dig-law compliance
assistant. You help excavation crews and utility workers understand state-specific
underground utility notification requirements before digging.

CRITICAL: Every answer about notification requirements must cite the specific state
law or regulatory reference. If you are uncertain about a jurisdiction's rules, say
"I need to verify this for [state] — call 811 directly."

Never improvise dig clearance procedures. A wrong answer here can kill someone.`,
  },

  askbuddy: {
    productId: 'askbuddy',
    model: 'claude-haiku-4-5',      // QR-code kiosk — needs sub-second first token
    maxTokens: 256,
    temperature: 0.3,
    stream: true,
    systemPrompt: `You are AskBuddy, a concise field reference assistant accessed
via QR-code kiosk. Answer in 2-4 sentences maximum. The user is standing at a
worksite and needs the answer now. If a question requires a longer answer, give
the key point first, then offer to expand.`,
  },

  guitarbuddy: {
    productId: 'guitarbuddy',
    model: 'claude-haiku-4-5',
    maxTokens: 768,
    temperature: 0.7,
    stream: true,
    systemPrompt: `You are GuitarBuddy, a music and chord assistant for a guitar
karaoke app. Help players with chord fingerings, song key transpositions, strumming
patterns, and music theory questions. Keep answers conversational and encouraging.
Not every user is a musician — many are just trying to play along to songs they love.`,
  },

  poolandspabuddy: {
    productId: 'poolandspabuddy',
    model: 'claude-sonnet-4-5',
    maxTokens: 1024,
    temperature: 0.1,
    stream: true,
    systemPrompt: `You are PoolAndSpaBuddy, a water chemistry assistant for pool and
spa owners. Interpret LSI (Langelier Saturation Index) readings, chemical test
results, and sensor data. When recommending chemical adjustments, always calculate
dose per 1,000 gallons and warn about chemical compatibility. Incorrect pool
chemistry causes equipment damage and health hazards — be precise.`,
  },

  // Add buddies here. The endpoint code does not change.
};
```

### The Universal Endpoint

```typescript
// app/api/chat/route.ts  (Next.js App Router — runs on Vercel Edge)

import Anthropic from '@anthropic-ai/sdk';
import { BUDDY_CONFIGS } from '@/lib/buddy-configs';

const client = new Anthropic({
  apiKey: process.env.ANTHROPIC_API_KEY,
});

export const runtime = 'edge';

export async function POST(req: Request) {
  const { productId, messages, context } = await req.json();

  // --- Validate product ---
  const config = BUDDY_CONFIGS[productId];
  if (!config) {
    return new Response(
      JSON.stringify({ error: `Unknown product: ${productId}` }),
      { status: 400, headers: { 'Content-Type': 'application/json' } }
    );
  }

  // --- Build system prompt ---
  // `context` is a freeform object the client sends: tenant name, user role,
  // active asset VIN, session flags, etc. Products declare what they use.
  const systemPrompt = context
    ? `${config.systemPrompt}\n\n[Session context]\n${JSON.stringify(context, null, 2)}`
    : config.systemPrompt;

  // --- Non-streaming path (batch jobs, test harnesses) ---
  if (!config.stream) {
    const response = await client.messages.create({
      model: config.model,
      max_tokens: config.maxTokens,
      temperature: config.temperature,
      system: systemPrompt,
      messages,
      ...(config.tools ? { tools: config.tools } : {}),
    });

    return new Response(JSON.stringify(response), {
      headers: { 'Content-Type': 'application/json' },
    });
  }

  // --- Streaming SSE path ---
  const encoder = new TextEncoder();

  const stream = new ReadableStream({
    async start(controller) {
      try {
        const anthropicStream = client.messages.stream({
          model: config.model,
          max_tokens: config.maxTokens,
          temperature: config.temperature,
          system: systemPrompt,
          messages,
          ...(config.tools ? { tools: config.tools } : {}),
        });

        // Pass raw Anthropic SSE event format through.
        // Clients use the Anthropic streaming helpers on the other end.
        for await (const chunk of anthropicStream) {
          const data = `data: ${JSON.stringify(chunk)}\n\n`;
          controller.enqueue(encoder.encode(data));
        }

        controller.enqueue(encoder.encode('data: [DONE]\n\n'));
        controller.close();

      } catch (err) {
        // Surface errors as SSE events so the client doesn't hang
        const errorPayload = {
          type: 'error',
          error: err instanceof Error ? err.message : 'Unknown error',
          productId,
        };
        controller.enqueue(
          encoder.encode(`data: ${JSON.stringify(errorPayload)}\n\n`)
        );
        controller.close();
      }
    },
  });

  return new Response(stream, {
    headers: {
      'Content-Type': 'text/event-stream',
      'Cache-Control': 'no-cache',
      'Connection': 'keep-alive',
      // These headers let your logging layer identify which product generated
      // which request without parsing the body
      'X-Buddy-Product': productId,
      'X-Buddy-Model': config.model,
    },
  });
}
```

### Client-Side Hook (React)

```typescript
// hooks/useBuddyChat.ts

import { useState, useCallback } from 'react';

interface Message {
  role: 'user' | 'assistant';
  content: string;
}

export function useBuddyChat(productId: string) {
  const [messages, setMessages] = useState<Message[]>([]);
  const [streaming, setStreaming] = useState(false);
  const [error, setError] = useState<string | null>(null);

  const sendMessage = useCallback(async (
    userInput: string,
    context?: Record<string, unknown>
  ) => {
    setError(null);

    const updatedMessages: Message[] = [
      ...messages,
      { role: 'user', content: userInput },
    ];

    // Optimistically append empty assistant bubble
    setMessages([...updatedMessages, { role: 'assistant', content: '' }]);
    setStreaming(true);

    try {
      const res = await fetch('/api/chat', {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ productId, messages: updatedMessages, context }),
      });

      if (!res.body) throw new Error('No response body');

      const reader = res.body.getReader();
      const decoder = new TextDecoder();
      let assistantText = '';

      while (true) {
        const { done, value } = await reader.read();
        if (done) break;

        const lines = decoder.decode(value).split('\n');
        for (const line of lines) {
          if (!line.startsWith('data: ') || line === 'data: [DONE]') continue;

          const chunk = JSON.parse(line.slice(6));

          if (chunk.type === 'error') {
            setError(chunk.error);
            break;
          }

          if (
            chunk.type === 'content_block_delta' &&
            chunk.delta?.type === 'text_delta'
          ) {
            assistantText += chunk.delta.text;
            setMessages(prev => {
              const next = [...prev];
              next[next.length - 1] = {
                role: 'assistant',
                content: assistantText,
              };
              return next;
            });
          }
        }
      }
    } catch (err) {
      setError(err instanceof Error ? err.message : 'Request failed');
    } finally {
      setStreaming(false);
    }
  }, [messages, productId]);

  const clearMessages = useCallback(() => setMessages([]), []);

  return { messages, streaming, error, sendMessage, clearMessages };
}
```

---

## Why It Works

- **Config is data, not code.** Adding product #13 means adding a config block — zero new API surface, zero new deploy, zero new transport-layer tests needed. The product team owns the config. The platform team owns the endpoint. No coordination required.
- **Model selection is explicit per product.** DigBuddy and AskBuddy run Haiku because they're high-volume, latency-sensitive, and the answer space is narrow. GarageBuddy runs Sonnet because OBD diagnosis benefits from stronger reasoning over noisy sensor data. You're not paying Sonnet prices for every QR-code kiosk lookup.
- **One cold start, all products.** Vercel Edge functions stay warm. One endpoint servicing 12 products means one warm instance, not 12 independent cold-start risks degrading your p95 latency.
- **Errors surface as SSE events, not hangs.** The `try/catch` inside the stream emits a typed error event before closing. The client always gets a `data:` line back. No silent timeouts that look like successful empty responses.

---

## Gotchas

**1. Don't put secrets in the static config registry.**
The `systemPrompt` field ends up in the edge function memory artifact at deploy time. If you're injecting tenant-specific credentials, internal system hostnames, or confidential operational thresholds into the prompt, pull those at request time from the tenant config table — not from a static registry bundled into your build. Static configs are fine for public product identity. Anything client-specific or confidential belongs in the DB, fetched per-request.

**2. Temperature 0 is not "safe" — it's deterministic.**
DigBuddy and PoolAndSpaBuddy use `temperature: 0` for procedures. This means if the model is wrong, it will be wrong the same way, every time, with full confidence. Low temperature without grounding is just consistent hallucination. Always pair `temperature: 0` configs with a RAG retrieval layer (see `citations-rag.md`) so the model is constrained to documented source material, not a confident prior from training data.

**3. The `messages` array grows unbounded across long sessions.**
This hook sends the full conversation history on every turn. For AskBuddy — stateless QR-code kiosk, one or two turns — that's fine. For GarageBuddy diagnostic sessions that can run 20+ exchanges across multiple DTCs, you'll eventually hit context limits or accumulate meaningful input token cost. Implement a sliding window: keep the last N turns, or summarize older context into a `[Previous context summary]` block prepended to the window. Do it before you need it, not after a session blows up at turn 23.

---

## Real-World Note

This pattern currently drives GarageBuddy, DigBuddy, AskBuddy, GuitarBuddy, PoolAndSpaBuddy, and 7 internal platform buddies (FieldBuddy, SafetyBuddy, InventoryBuddy, and others) off a single Vercel deployment at `buddysuite.vercel.app`. Total endpoint count: 1. The product count has grown from 3 to 12+ without touching the transport layer once. When Anthropic released a new Haiku version, the model strings in the config got bumped in one commit — every Haiku-class buddy updated atomically with no risk of partial deployment.
