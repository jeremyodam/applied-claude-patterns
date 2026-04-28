# Universal Streaming Endpoint

## Problem

You are building multiple AI-powered products — field ops assistant, safety assistant, fleet assistant, scheduling assistant. Each needs a Claude backend. The naive approach is one API route per product. That means duplicated auth, duplicated streaming logic, duplicated error handling, and a maintenance nightmare when Anthropic ships a new model.

The better architecture: one endpoint that dispatches to any product by config.

## Pattern

A single serverless function receives a `buddy` parameter alongside the user message. It looks up the system prompt, model, and RAG config for that product from a config store, then streams a Claude response back. The calling client never knows it shares infrastructure with nineteen other products.

```typescript
// api/claude.ts — universal dispatch endpoint
import Anthropic from "@anthropic-ai/sdk";
import { getBuddyConfig } from "../lib/config";
import { retrieveContext } from "../lib/rag";

const client = new Anthropic();

export async function POST(req: Request) {
  const { buddy, message, conversationHistory = [], clientId } = await req.json();

  // Load product config — system prompt, model, RAG toggle, max tokens
  const config = await getBuddyConfig(buddy, clientId);
  if (!config) {
    return new Response("Unknown product", { status: 400 });
  }

  // Optionally augment system prompt with retrieved context
  let systemPrompt = config.systemPrompt;
  if (config.ragEnabled) {
    const context = await retrieveContext(buddy, message, clientId);
    if (context) {
      systemPrompt += `\n\n<retrieved_context>\n${context}\n</retrieved_context>`;
    }
  }

  // Build message history
  const messages = [
    ...conversationHistory,
    { role: "user" as const, content: message },
  ];

  // Stream response
  const encoder = new TextEncoder();
  const stream = new ReadableStream({
    async start(controller) {
      try {
        const claudeStream = await client.messages.stream({
          model: config.model ?? "claude-sonnet-4-5",
          max_tokens: config.maxTokens ?? 1024,
          system: systemPrompt,
          messages,
        });

        for await (const chunk of claudeStream) {
          if (
            chunk.type === "content_block_delta" &&
            chunk.delta.type === "text_delta"
          ) {
            controller.enqueue(encoder.encode(chunk.delta.text));
          }
        }
      } catch (err) {
        controller.error(err);
      } finally {
        controller.close();
      }
    },
  });

  return new Response(stream, {
    headers: {
      "Content-Type": "text/plain; charset=utf-8",
      "Transfer-Encoding": "chunked",
    },
  });
}
```

## Config Shape

```typescript
// lib/config.ts
export interface BuddyConfig {
  systemPrompt: string;
  model?: string;
  maxTokens?: number;
  ragEnabled: boolean;
  ragNamespace?: string; // scopes vector retrieval per product + client
}

export async function getBuddyConfig(
  buddy: string,
  clientId: string
): Promise<BuddyConfig | null> {
  // Load from Supabase, Redis, or flat config files depending on your scale
  // Row-level security on clientId ensures cross-tenant isolation at the data layer
  const { data } = await supabase
    .from("module_configs")
    .select("*")
    .eq("buddy", buddy)
    .eq("client_org_id", clientId)
    .single();

  return data ?? null;
}
```

## Client-Side Streaming

```typescript
// Minimal streaming client — same pattern for every product
async function streamBuddy(buddy: string, message: string) {
  const res = await fetch("/api/claude", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({ buddy, message, conversationHistory }),
  });

  const reader = res.body!.getReader();
  const decoder = new TextDecoder();
  let output = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;
    output += decoder.decode(value);
    setResponse(output); // stream into UI as it arrives
  }
}
```

## Why This Works at Scale

- **Zero duplication.** Auth, rate limiting, error handling, and model upgrades live in one place.
- **Config-driven rollout.** Add a new product by inserting a row, not deploying new code.
- **Tenant isolation.** `clientId` scopes every config lookup and every RAG retrieval. One client's system prompt never bleeds into another's.
- **Model portability.** Swap `claude-sonnet-4-5` to `claude-opus-4-7` in config. No code changes.

## Tradeoffs

Cold starts on serverless can add 200-400ms to the first token. For interactive field tools this is acceptable. For sub-100ms SLAs, move to a persistent runtime (Fly.io, Railway) and keep the same dispatch logic.
