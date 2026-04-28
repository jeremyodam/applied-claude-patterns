# RAG with Native Citations

## Problem

Field operators asking an AI assistant about safety procedures need to trust the answer. "According to the model" is not good enough. They need to know which document, which section, which page. Hallucinated procedures in regulated operations are a liability.

Anthropic's Native Citations API solves this at the model level — Claude identifies exactly which source chunks support each claim in its response. No prompt engineering required. No post-processing regex to extract fake citations.

## Pattern

Retrieve relevant document chunks from a vector store, pass them as `document` sources in the API call, and let Claude cite them natively. Surface the citations in the UI alongside the answer.

## Ingestion Pipeline

```python
# ingest.py — chunk documents and store embeddings
import anthropic
from supabase import create_client
from pypdf import PdfReader

client = anthropic.Anthropic()
supabase = create_client(SUPABASE_URL, SUPABASE_KEY)

def chunk_document(pdf_path: str, chunk_size: int = 1500, overlap: int = 200):
    reader = PdfReader(pdf_path)
    chunks = []
    buffer = ""

    for page_num, page in enumerate(reader.pages):
        text = page.extract_text()
        buffer += f"\n[Page {page_num + 1}]\n{text}"

        while len(buffer) >= chunk_size:
            chunk = buffer[:chunk_size]
            chunks.append({
                "text": chunk,
                "page": page_num + 1,
                "source": pdf_path,
            })
            buffer = buffer[chunk_size - overlap:]

    if buffer.strip():
        chunks.append({"text": buffer, "source": pdf_path})

    return chunks

def embed_and_store(chunks: list, namespace: str):
    for chunk in chunks:
        # Anthropic embeddings via voyage-3 or use sentence-transformers locally
        embedding = get_embedding(chunk["text"])

        supabase.table("document_chunks").insert({
            "content": chunk["text"],
            "embedding": embedding,
            "source": chunk["source"],
            "page": chunk.get("page"),
            "namespace": namespace,  # scopes retrieval per product/client
        }).execute()
```

## Retrieval

```typescript
// lib/rag.ts
export async function retrieveContext(
  namespace: string,
  query: string,
  clientId: string,
  topK: number = 5
): Promise<RetrievedChunk[]> {
  const queryEmbedding = await embed(query);

  // pgvector cosine similarity search — RLS on clientId enforced at DB level
  const { data } = await supabase.rpc("match_chunks", {
    query_embedding: queryEmbedding,
    match_namespace: namespace,
    match_client_id: clientId,
    match_count: topK,
  });

  return data ?? [];
}
```

```sql
-- Supabase function for HNSW vector search
CREATE OR REPLACE FUNCTION match_chunks(
  query_embedding vector(1536),
  match_namespace text,
  match_client_id text,
  match_count int
)
RETURNS TABLE (id uuid, content text, source text, page int, similarity float)
LANGUAGE sql STABLE
AS $$
  SELECT id, content, source, page,
    1 - (embedding <=> query_embedding) AS similarity
  FROM document_chunks
  WHERE namespace = match_namespace
    AND client_org_id = match_client_id
  ORDER BY embedding <=> query_embedding
  LIMIT match_count;
$$;
```

## API Call with Native Citations

```typescript
// Pass retrieved chunks as document sources — Claude cites them natively
export async function askWithCitations(
  query: string,
  chunks: RetrievedChunk[],
  systemPrompt: string
) {
  const sources = chunks.map((chunk, i) => ({
    type: "document" as const,
    source: {
      type: "text" as const,
      media_type: "text/plain" as const,
      data: chunk.content,
    },
    title: `${chunk.source} — Page ${chunk.page}`,
    citations: { enabled: true },
  }));

  const response = await client.messages.create({
    model: "claude-sonnet-4-5",
    max_tokens: 1024,
    system: systemPrompt,
    messages: [
      {
        role: "user",
        content: [
          ...sources,
          { type: "text", text: query },
        ],
      },
    ],
  });

  return response;
}
```

## Rendering Citations in the UI

```typescript
// Extract citations from response content blocks
function renderResponseWithCitations(content: ContentBlock[]) {
  return content.map((block) => {
    if (block.type === "text") {
      return <p key={block.id}>{block.text}</p>;
    }

    if (block.type === "tool_result") return null;

    // Citation blocks reference source documents by index
    if ("citations" in block && block.citations) {
      return block.citations.map((cite) => (
        <span key={cite.start_char} className="citation">
          [{cite.document_title} · p.{cite.page}]
        </span>
      ));
    }
  });
}
```

## Why Native Citations Over Prompt Engineering

Prompt-engineered citations ("always cite your sources") produce hallucinated references. The model makes up plausible-sounding document names and page numbers when the real source isn't clearly present in context.

Native Citations works differently — Claude only cites chunks that were actually passed in the `documents` array. It cannot fabricate a citation to a document it wasn't given. For regulated operations (NFPA 54, NFPA 58, 49 CFR 192, company SOPs), this is the difference between a useful tool and a liability.

## Tradeoffs

Passing large document chunks increases input token cost. For latency-sensitive paths, cap `topK` at 3-5 and tune chunk size to 1000-1500 tokens. For compliance-critical workflows where missing context is worse than slower responses, go higher.
