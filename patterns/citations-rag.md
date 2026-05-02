# Pattern: RAG with Native Anthropic Citations

## Problem

Workers in regulated environments asking an AI assistant about safety procedures need to trust the answer. "The model thinks" is not an acceptable source when the procedure involves excavation near a live gas main or chemical dosing near a public pool. They need to know which document, which section, which page number. Traditional approaches involve prompt engineering ("always cite your sources") or post-processing regex to extract source references — both produce hallucinated citations. The model generates plausible-sounding document names and page numbers that don't exist, because nothing in the prompt-engineering approach actually constrains the model to the provided source material.

Anthropic's native Citations API solves this at the model level. Claude identifies exactly which source chunks support each claim in its response and embeds citation metadata directly in the content blocks. It cannot fabricate a citation to a document it wasn't given — the citation index references only the documents passed in the request. For regulated operations where a wrong procedure is a liability, this is the difference between a useful tool and a lawsuit.

---

## Solution

### Schema: Document Chunks with pgvector

```sql
-- Supabase — document_chunks table with HNSW index
CREATE EXTENSION IF NOT EXISTS vector;

CREATE TABLE document_chunks (
  id           uuid PRIMARY KEY DEFAULT gen_random_uuid(),
  content      text NOT NULL,
  embedding    vector(1536),            -- dimension matches your embedding model
  source       text NOT NULL,           -- file name or URL
  page         int,                     -- page number if applicable
  section      text,                    -- section title if extractable
  namespace    text NOT NULL,           -- scopes retrieval to product + client
  client_org_id text NOT NULL,          -- enforced by RLS
  created_at   timestamptz DEFAULT now()
);

-- HNSW index for fast approximate nearest-neighbor search
-- ef_construction=128 is the Supabase recommended starting point.
-- Tune m and ef_construction after you have real query patterns.
CREATE INDEX document_chunks_embedding_idx
  ON document_chunks
  USING hnsw (embedding vector_cosine_ops)
  WITH (m = 16, ef_construction = 128);

-- RLS: clients only retrieve their own chunks
ALTER TABLE document_chunks ENABLE ROW LEVEL SECURITY;

CREATE POLICY "tenant_isolation" ON document_chunks
  FOR ALL USING (client_org_id = current_setting('app.client_org_id'));

-- Retrieval function — returns top-k chunks by cosine similarity
CREATE OR REPLACE FUNCTION match_chunks(
  query_embedding  vector(1536),
  match_namespace  text,
  match_client_id  text,
  match_count      int DEFAULT 5,
  similarity_threshold float DEFAULT 0.7
)
RETURNS TABLE (
  id         uuid,
  content    text,
  source     text,
  page       int,
  section    text,
  similarity float
)
LANGUAGE sql STABLE AS $$
  SELECT
    id,
    content,
    source,
    page,
    section,
    1 - (embedding <=> query_embedding) AS similarity
  FROM document_chunks
  WHERE
    namespace      = match_namespace
    AND client_org_id = match_client_id
    AND 1 - (embedding <=> query_embedding) >= similarity_threshold
  ORDER BY embedding <=> query_embedding
  LIMIT match_count;
$$;
```

### Ingestion Pipeline

```typescript
// scripts/ingest-document.ts
// Run this when a new procedure manual is uploaded.

import Anthropic from '@anthropic-ai/sdk';
import { createClient } from '@supabase/supabase-js';
import { PdfReader } from 'pdfreader';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
const supabase  = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_SERVICE_KEY!   // service key for ingestion — not used in app
);

interface Chunk {
  text: string;
  source: string;
  page?: number;
  section?: string;
}

async function chunkPdf(
  pdfPath: string,
  chunkSize = 1500,
  overlap = 200
): Promise<Chunk[]> {
  // Extract text per page, then window into chunks
  const pages: { pageNum: number; text: string }[] = await extractPdfPages(pdfPath);
  const chunks: Chunk[] = [];
  let buffer = '';
  let currentPage = 1;

  for (const { pageNum, text } of pages) {
    buffer += `\n[Page ${pageNum}]\n${text}`;
    currentPage = pageNum;

    while (buffer.length >= chunkSize) {
      const chunkText = buffer.slice(0, chunkSize);
      chunks.push({ text: chunkText, source: pdfPath, page: currentPage });
      buffer = buffer.slice(chunkSize - overlap);
    }
  }

  if (buffer.trim()) {
    chunks.push({ text: buffer, source: pdfPath, page: currentPage });
  }

  return chunks;
}

async function embedChunk(text: string): Promise<number[]> {
  // Using voyage-3 via Anthropic for embedding consistency
  // Replace with text-embedding-3-small if you're already on OpenAI
  const response = await fetch('https://api.voyageai.com/v1/embeddings', {
    method: 'POST',
    headers: {
      'Authorization': `Bearer ${process.env.VOYAGE_API_KEY}`,
      'Content-Type': 'application/json',
    },
    body: JSON.stringify({ model: 'voyage-3', input: text }),
  });
  const data = await response.json();
  return data.data[0].embedding;
}

export async function ingestDocument(
  pdfPath: string,
  namespace: string,
  clientOrgId: string
): Promise<void> {
  const chunks = await chunkPdf(pdfPath);
  console.log(`Ingesting ${chunks.length} chunks for ${namespace}`);

  for (const chunk of chunks) {
    const embedding = await embedChunk(chunk.text);

    await supabase.from('document_chunks').insert({
      content:       chunk.text,
      embedding,
      source:        chunk.source,
      page:          chunk.page,
      namespace,
      client_org_id: clientOrgId,
    });
  }

  console.log(`Done ingesting ${pdfPath}`);
}

// Example: ingest the DigBuddy 811 compliance manual for a new client
// await ingestDocument('./docs/wa-811-procedures.pdf', 'acme-corp:digbuddy', 'acme-corp');
```

### Retrieval + Citations API Call

```typescript
// lib/rag-citations.ts

import Anthropic from '@anthropic-ai/sdk';
import { createClient } from '@supabase/supabase-js';

const anthropic = new Anthropic({ apiKey: process.env.ANTHROPIC_API_KEY });
const supabase  = createClient(
  process.env.SUPABASE_URL!,
  process.env.SUPABASE_ANON_KEY!
);

export interface RetrievedChunk {
  id: string;
  content: string;
  source: string;
  page: number | null;
  section: string | null;
  similarity: number;
}

export async function retrieveContext(
  namespace: string,
  query: string,
  clientOrgId: string,
  topK = 5
): Promise<RetrievedChunk[]> {
  const queryEmbedding = await embedChunk(query);

  // RLS enforces clientOrgId at the DB level — this filter is defense-in-depth
  const { data, error } = await supabase.rpc('match_chunks', {
    query_embedding:    queryEmbedding,
    match_namespace:    namespace,
    match_client_id:    clientOrgId,
    match_count:        topK,
    similarity_threshold: 0.7,
  });

  if (error) throw new Error(`Retrieval failed: ${error.message}`);
  return data ?? [];
}

export async function askWithCitations(
  query: string,
  chunks: RetrievedChunk[],
  systemPrompt: string,
  model = 'claude-sonnet-4-5'
): Promise<Anthropic.Message> {
  // Map retrieved chunks to Anthropic document sources.
  // The `citations: { enabled: true }` flag tells Claude to cite these documents
  // natively — not via prompt instruction, but at the API level.
  const documentSources: Anthropic.DocumentBlockParam[] = chunks.map((chunk) => ({
    type: 'document',
    source: {
      type: 'text',
      media_type: 'text/plain',
      data: chunk.content,
    },
    title: [
      chunk.source,
      chunk.page    ? `Page ${chunk.page}`    : null,
      chunk.section ? `§ ${chunk.section}`    : null,
    ].filter(Boolean).join(' — '),
    citations: { enabled: true },
  }));

  return anthropic.messages.create({
    model,
    max_tokens: 1024,
    system: systemPrompt,
    messages: [
      {
        role: 'user',
        content: [
          // Documents come before the query — Claude reads sources, then answers
          ...documentSources,
          { type: 'text', text: query },
        ],
      },
    ],
  });
}
```

### Rendering Citations in the UI

```typescript
// components/CitedResponse.tsx

import Anthropic from '@anthropic-ai/sdk';

interface Props {
  content: Anthropic.ContentBlock[];
}

export function CitedResponse({ content }: Props) {
  return (
    <div className="cited-response">
      {content.map((block, i) => {
        if (block.type !== 'text') return null;

        // Claude returns text blocks with embedded citation spans
        // when citations are present, the block includes a citations array
        const textBlock = block as Anthropic.TextBlock & {
          citations?: Array<{
            document_title: string;
            start_char: number;
            end_char: number;
          }>;
        };

        return (
          <div key={i} className="response-block">
            <p>{textBlock.text}</p>
            {textBlock.citations && textBlock.citations.length > 0 && (
              <div className="citations">
                {textBlock.citations.map((cite, j) => (
                  <span key={j} className="citation-badge">
                    {cite.document_title}
                  </span>
                ))}
              </div>
            )}
          </div>
        );
      })}
    </div>
  );
}
```

### Full Request Flow (Composed)

```typescript
// app/api/chat-rag/route.ts

import { retrieveContext, askWithCitations } from '@/lib/rag-citations';
import { getModuleConfig } from '@/lib/config';

export async function POST(req: Request) {
  const { query, module, clientOrgId } = await req.json();

  const config    = await getModuleConfig(clientOrgId, module);
  const namespace = config.rag_namespace ?? `${clientOrgId}:${module}`;

  // 1. Retrieve relevant chunks from pgvector
  const chunks = await retrieveContext(namespace, query, clientOrgId, 5);

  if (chunks.length === 0) {
    return Response.json({
      answer: "I don't have documentation to answer that. Please contact your supervisor.",
      citations: [],
    });
  }

  // 2. Pass chunks to Claude with native citations enabled
  const response = await askWithCitations(
    query,
    chunks,
    config.system_prompt,
    config.model
  );

  return Response.json(response);
}
```

---

## Why It Works

- **Citations are grounded, not generated.** Claude can only cite documents that were passed in the `documents` array of the request. If the source isn't there, there's no citation — not a made-up one. The index is structural, not semantic. This is the critical difference from prompt-engineered citations.
- **Similarity threshold filters noise before it hits the model.** The `similarity_threshold: 0.7` in the pgvector query means low-relevance chunks never reach the API call. Passing garbage context produces garbage-confident answers. Raise the threshold for high-stakes queries; lower it for exploratory ones.
- **Namespace scoping keeps retrieval product- and client-specific.** The `namespace` field on every chunk means DigBuddy's 811 documents never appear in a GarageBuddy diagnostic response. Combined with RLS, this gives you two independent isolation layers: one at the application level, one at the database level.
- **The "no chunks" path is explicit.** When retrieval returns zero results above threshold, the system returns a known-safe response rather than letting the model hallucinate an answer from training data. For regulated procedures, a "I don't have documentation on that" is always better than a confident wrong answer.

---

## Gotchas

**1. Chunk size directly trades recall for precision.**
Chunks of 500 tokens retrieve narrow, precise passages — good for procedural lookups where the answer is one step. Chunks of 2000 tokens retrieve broad context — better for conceptual questions where multiple sentences contribute. Start at 1200-1500 with 200-token overlap and measure citation quality on your actual queries before tuning. The wrong chunk size causes the model to cite the right document but the wrong passage.

**2. HNSW indexes don't update in-place.**
Supabase pgvector builds the HNSW index at creation time. New rows added after index creation are included in subsequent scans, but the index quality degrades as the table grows without reindexing. Schedule a `REINDEX INDEX CONCURRENTLY document_chunks_embedding_idx` monthly, or after bulk ingestion events. If you skip this, retrieval quality silently degrades as your corpus grows — you won't see an error, you'll just see worse citations.

**3. The citations API is not available on streaming responses.**
As of mid-2025, native citations require a non-streaming `messages.create` call. If you need citations in a streaming UX, you have two options: (a) use non-streaming for the RAG path and streaming only for conversational paths that don't require citations, or (b) return the full response and stream it to the client from your own endpoint after the fact. For safety-procedure use cases, the latency tradeoff is worth it — a cited answer that takes 3 seconds beats an uncited answer in 1 second.

---

## Real-World Note

This pattern is the core retrieval layer for DigBuddy (811 dig-law compliance) and InventoryBuddy (utility tool calibration records). In both cases, the answer isn't useful without the source — field workers need to show auditors exactly which document they followed. Native citations replace a custom post-processing layer that was adding 200ms of latency and occasionally generating malformed citation strings. The API-level citations are structurally correct every time, and the UI renders them as clickable document badges linked to the source PDF.
