# Skill: AI Agent Design

> ⚠️ Đồng bộ ARCHITECTURE_CONTRACT 29/5/2026: PK=uuid, tenant_id=uuid, TS camelCase, RBAC dấu `.`. Xem docs/ARCHITECTURE_CONTRACT.md.

> Khi nào load: Định nghĩa AI agent mới, tích hợp Claude/Gemini API, build RAG

---

## Decision tree: chọn loại AI

```
Task gì?
├── Phân loại đơn giản (intent, sentiment) → Haiku 4.5
├── Phân tích văn bản phức tạp, generate content → Sonnet 4.6
├── Reasoning đa bước, ra quyết định kinh doanh → Opus 4.7
├── Image/Video understanding → Gemini 2.5 Flash (rẻ) or Claude Vision (cao cấp)
├── Embedding cho search/RAG → Gemini text-embedding-004 (768-dim)
└── Local inference (privacy, cost) → Ollama (Qwen, Llama, Mistral)
```

---

## Agent definition (BẮT BUỘC theo template)

Xem [`rules/13-ai-agents.md#agent-definition-template`](../../rules/13-ai-agents.md).

---

## Anti-hallucination prompting

### Rule 1: Structured output
```
❌ "Phân tích tin nhắn này"
✅ "Trả JSON với schema: { category, confidence, entities }"
```

### Rule 2: Cho ví dụ rõ ràng
```
Input: "Cho mình 2 ly trà sữa size M"
Output:
{
  "category": "order_request",
  "confidence": 0.95,
  "entities": {
    "products": [{"name": "trà sữa", "size": "M", "quantity": 2}]
  }
}
```

### Rule 3: Cấm đoán
```
QUY TẮC:
- KHÔNG bịa thông tin không có trong input
- Confidence < 0.85 → category = "uncertain", giải thích trong reasoning
- Entity không thấy → để null, KHÔNG gán default
```

### Rule 4: Yêu cầu reasoning
```
Trả về thêm field `reasoning` giải thích vì sao chọn category này.
Reasoning sẽ được dùng để debug + train lại agent.
```

---

## RAG implementation

### Schema
```sql
-- Knowledge base (uuid PK + uuid tenant_id theo ARCHITECTURE_CONTRACT mục 2+3)
CREATE TABLE knowledge_chunks (
  id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
  tenant_id UUID NOT NULL,
  source_id UUID NOT NULL,           -- Document ID
  source_type TEXT NOT NULL,         -- 'sop', 'menu', 'faq', 'policy'
  chunk_index INT NOT NULL,
  content TEXT NOT NULL,
  metadata JSONB,                    -- { title, section, page, ... }
  embedding vector(768),             -- Gemini text-embedding-004
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX knowledge_chunks_embedding_idx 
  ON knowledge_chunks USING hnsw (embedding vector_cosine_ops);

CREATE INDEX knowledge_chunks_tenant_source_idx 
  ON knowledge_chunks (tenant_id, source_type, source_id);
```

### Indexing pipeline
```typescript
async function indexDocument(doc: Document) {
  // 1. Chunk (200-500 tokens, overlap 50)
  const chunks = splitIntoChunks(doc.content, { size: 400, overlap: 50 });
  
  // 2. Embed batch
  const embeddings = await gemini.embedBatch(chunks.map(c => c.text));
  
  // 3. Insert
  await db.insert(knowledgeChunks).values(
    chunks.map((chunk, i) => ({
      // id auto-generated via defaultRandom() theo CONTRACT mục 2
      tenantId: doc.tenantId,
      sourceId: doc.id,
      sourceType: doc.type,
      chunkIndex: i,
      content: chunk.text,
      metadata: { ...chunk.metadata, docTitle: doc.title },
      embedding: embeddings[i],
    }))
  );
}
```

### Retrieval
```typescript
async function retrieveContext(query: string, tenantId: string, k = 5) {
  const queryEmbedding = await gemini.embed(query);
  
  return db.execute(sql`
    SELECT id, content, metadata,
      1 - (embedding <=> ${queryEmbedding}::vector) AS similarity
    FROM knowledge_chunks
    WHERE tenant_id = ${tenantId}
    AND 1 - (embedding <=> ${queryEmbedding}::vector) > 0.7
    ORDER BY embedding <=> ${queryEmbedding}::vector
    LIMIT ${k}
  `);
}
```

### Generation với context
```typescript
async function answerWithRAG(question: string, tenantId: string) {
  const context = await retrieveContext(question, tenantId);
  
  const systemPrompt = `
Bạn là trợ lý của WECHA. Trả lời câu hỏi DỰA TRÊN context dưới đây.

QUY TẮC:
- Nếu context không đủ thông tin → "Tôi chưa có thông tin về vấn đề này, xin chuyển cho nhân viên hỗ trợ."
- KHÔNG bịa thông tin ngoài context
- Trích dẫn nguồn ở cuối câu trả lời

Context:
${context.map((c, i) => `[${i + 1}] ${c.content}`).join('\n\n')}
`;
  
  const response = await claude.messages.create({
    model: 'claude-haiku-4-5-20251001',
    max_tokens: 500,
    system: systemPrompt,
    messages: [{ role: 'user', content: question }],
  });
  
  // Log cho cost tracking
  await trackCost('rag-qa-v1', response);
  
  return response.content[0].text;
}
```

---

## Agent evaluation framework

```typescript
interface EvaluationCase {
  id: string;
  input: any;
  expectedOutput: any;
  expectedConfidenceMin: number;
}

async function evaluateAgent(agentId: string, cases: EvaluationCase[]) {
  const results = await Promise.all(cases.map(async (c) => {
    const actual = await runAgent(agentId, c.input);
    
    return {
      caseId: c.id,
      pass: deepEqual(actual.output, c.expectedOutput) && actual.confidence >= c.expectedConfidenceMin,
      actual,
      expected: { output: c.expectedOutput, confidence: c.expectedConfidenceMin },
    };
  }));
  
  const accuracy = results.filter(r => r.pass).length / results.length;
  
  return {
    agentId,
    totalCases: cases.length,
    passed: results.filter(r => r.pass).length,
    accuracy,
    failures: results.filter(r => !r.pass),
  };
}

// Run weekly cron
@Cron('0 0 * * 0')
async weeklyAgentEval() {
  for (const agent of getAllAgents()) {
    const cases = await loadEvalCases(agent.id);
    const result = await evaluateAgent(agent.id, cases);
    
    if (result.accuracy < 0.9) {
      await alertOps('agent_accuracy_drop', { agentId: agent.id, accuracy: result.accuracy });
    }
  }
}
```

---

## Cost optimization

### Model routing
```typescript
async function routeToModel(task: Task): Promise<string> {
  if (task.type === 'classification' && task.complexity === 'low') {
    return 'claude-haiku-4-5-20251001';  // $0.80/M input
  }
  if (task.type === 'generation' && task.complexity === 'medium') {
    return 'claude-sonnet-4-6';  // $3/M input
  }
  if (task.requiresReasoning) {
    return 'claude-opus-4-7';  // $15/M input
  }
  return 'claude-haiku-4-5-20251001';
}
```

### Caching
```typescript
async function callWithCache(prompt: string, agentId: string): Promise<string> {
  const cacheKey = `agent:${agentId}:${hashPrompt(prompt)}`;
  
  const cached = await redis.get(cacheKey);
  if (cached) {
    await metrics.incr('agent_cache_hit', { agent_id: agentId });
    return cached;
  }
  
  const result = await callLLM(prompt);
  await redis.set(cacheKey, result, 'EX', 3600);
  
  return result;
}
```

### Batch processing
```typescript
// Thay vì 100 calls → 1 batch call
const results = await claude.messages.batches.create({
  requests: tasks.map(t => ({
    custom_id: t.id,
    params: { model: 'claude-haiku-4-5-20251001', messages: [...] },
  })),
});
// Save 50% chi phí
```

---

## Human-in-the-loop pattern

```typescript
async function aiWithHumanFallback(task: Task) {
  const aiResult = await runAgent(task.agentId, task.input);
  
  if (aiResult.confidence < 0.85) {
    // Escalate to human
    const humanTask = await db.insert(humanReviewTasks).values({
      // id auto-generated via defaultRandom() theo CONTRACT mục 2
      tenantId: task.tenantId,
      sourceType: 'agent',
      sourceId: task.agentId,
      taskType: task.type,
      priority: task.priority ?? 'medium',
      payload: task.input,
      context: {
        aiConfidence: aiResult.confidence,
        aiSuggestion: aiResult.output,
        aiReasoning: aiResult.reasoning,
      },
      status: 'pending',
      dueAt: new Date(Date.now() + 30 * 60_000),  // 30 phút
    }).returning();
    
    // Notify human
    await sendTelegram('support_team_chat', {
      title: 'New review task',
      payload: humanTask[0],
      url: `https://erp.wecha.vn/review/${humanTask[0].id}`,
    });
    
    return { status: 'pending_human_review', taskId: humanTask[0].id };
  }
  
  return { status: 'auto_completed', result: aiResult.output };
}
```

---

**END Skill: AI Agent Design**
