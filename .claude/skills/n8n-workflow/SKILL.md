# Skill: n8n Workflow

> Khi nào load: Tạo/sửa workflow n8n, tích hợp ERP API với n8n

---

## Mọi workflow PHẢI có

### 1. Naming convention
`[Domain] - [Action] - [Trigger]`

Ví dụ:
- `[Inventory] - Auto Reorder - Daily 6AM`
- `[Orders] - Sync Shopee - Webhook`
- `[Marketing] - Send Birthday Wish - Daily 8AM`
- `[HR] - Attendance Reminder - Daily 8:30AM`

### 2. Tags
- Environment: `production` / `staging` / `draft`
- Domain: `domain:orders` / `domain:inventory` / `domain:hr`
- Criticality: `criticality:high` / `medium` / `low`

### 3. Error handling
- Mọi node có khả năng fail → có Error Trigger
- Error → log vào Postgres `workflow_executions` + alert Telegram

### 4. Timeout
- Default 5 phút, có thể tăng nhưng phải có lý do

### 5. Idempotency
- Nếu trigger lại với cùng input → kết quả phải giống

### 6. Logging
- Mỗi step ghi vào `workflow_steps` table

### 7. Versioning
- Workflow JSON commit vào `apps/n8n/workflows/`

### 8. README per workflow

```markdown
# [Inventory] - Auto Reorder - Daily 6AM

## Mục đích
Mỗi ngày 6h sáng, check tồn kho. Nếu sản phẩm có tồn < min_stock thì tự tạo PO.

## Trigger
Schedule: 0 6 * * * (daily 6AM)

## Steps
1. Schedule trigger
2. Postgres: SELECT products WHERE stock < min_stock AND auto_reorder = true
3. IF empty → END
4. Loop products:
   a. Call ERP API: POST /purchase-orders/draft
   b. Send Telegram alert to procurement team
5. Log execution to workflow_executions

## Failure modes
- Postgres down → retry 3 lần, sau đó alert ops
- ERP API 500 → skip product, continue, log error
- Telegram fail → không block, chỉ log

## Owner
@trí (engineering@wecha.vn)

## Cost estimate
- 0 LLM call
- ~50 DB queries/day
- ~5 ERP API calls/day
- Free tier
```

---

## Workflow tracking schema

```sql
CREATE TABLE workflow_executions (
  id TEXT PRIMARY KEY,
  tenant_id TEXT NOT NULL,
  workflow_id TEXT NOT NULL,
  workflow_name TEXT NOT NULL,
  workflow_version INT NOT NULL,
  
  trigger_type TEXT NOT NULL,           -- 'webhook', 'cron', 'manual'
  trigger_payload JSONB,
  
  status TEXT NOT NULL,                  -- 'running', 'success', 'failed', 'partial'
  started_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
  finished_at TIMESTAMPTZ,
  duration_ms INT,
  
  total_steps INT,
  successful_steps INT,
  failed_steps INT,
  
  error TEXT,
  output JSONB,
  
  created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX workflow_exec_tenant_workflow_time_idx 
  ON workflow_executions (tenant_id, workflow_id, started_at DESC);

CREATE TABLE workflow_steps (
  id BIGSERIAL PRIMARY KEY,
  execution_id TEXT NOT NULL REFERENCES workflow_executions(id),
  step_index INT NOT NULL,
  step_name TEXT NOT NULL,
  step_type TEXT NOT NULL,               -- 'http', 'postgres', 'function', ...
  
  input JSONB,
  output JSONB,
  error TEXT,
  
  status TEXT NOT NULL,                  -- 'success', 'failed', 'skipped'
  started_at TIMESTAMPTZ NOT NULL,
  finished_at TIMESTAMPTZ,
  duration_ms INT
);
```

---

## n8n custom node template (TypeScript)

```typescript
// apps/n8n/custom-nodes/wecha-erp-api/WechaErpApi.node.ts
import {
  IExecuteFunctions,
  INodeExecutionData,
  INodeType,
  INodeTypeDescription,
} from 'n8n-workflow';

export class WechaErpApi implements INodeType {
  description: INodeTypeDescription = {
    displayName: 'WECHA ERP API',
    name: 'wechaErpApi',
    group: ['transform'],
    version: 1,
    description: 'Gọi WECHA ERP API với HMAC signing',
    defaults: { name: 'WECHA ERP API' },
    inputs: ['main'],
    outputs: ['main'],
    credentials: [{ name: 'wechaErpApi', required: true }],
    properties: [
      {
        displayName: 'Endpoint',
        name: 'endpoint',
        type: 'string',
        default: '/api/v1/orders',
        required: true,
      },
      {
        displayName: 'Method',
        name: 'method',
        type: 'options',
        options: [
          { name: 'GET', value: 'GET' },
          { name: 'POST', value: 'POST' },
          { name: 'PATCH', value: 'PATCH' },
        ],
        default: 'GET',
      },
      // ... more
    ],
  };
  
  async execute(this: IExecuteFunctions): Promise<INodeExecutionData[][]> {
    const credentials = await this.getCredentials('wechaErpApi');
    const baseUrl = credentials.baseUrl as string;
    const apiKey = credentials.apiKey as string;
    
    const items = this.getInputData();
    const returnData: INodeExecutionData[] = [];
    
    for (let i = 0; i < items.length; i++) {
      const endpoint = this.getNodeParameter('endpoint', i) as string;
      const method = this.getNodeParameter('method', i) as string;
      const body = items[i].json;
      
      // HMAC sign
      const timestamp = Math.floor(Date.now() / 1000);
      const signature = signRequest(method, endpoint, body, timestamp, apiKey);
      
      const response = await this.helpers.request({
        method,
        url: `${baseUrl}${endpoint}`,
        headers: {
          'Content-Type': 'application/json',
          'X-Wecha-Timestamp': timestamp,
          'X-Wecha-Signature': signature,
          'X-Idempotency-Key': `n8n-${this.getExecutionId()}-${i}`,
        },
        body,
        json: true,
      });
      
      returnData.push({ json: response });
    }
    
    return [returnData];
  }
}
```

---

## Common workflow patterns

### Pattern 1: Auto-sync e-commerce orders

```
[Webhook trigger from Shopee]
  ↓
[Validate signature]
  ↓
[Map Shopee → ERP format (Function node)]
  ↓
[POST /api/v1/orders (WECHA ERP node)]
  ↓
[IF success]
  → [Update Shopee status: confirmed]
  → [Log success]
  ↓
[IF fail]
  → [Send Telegram alert]
  → [Create human_review_task]
```

### Pattern 2: AI message classification

```
[Zalo OA webhook: incoming message]
  ↓
[Save to incoming_messages table]
  ↓
[Call AI agent: order-classifier-v1]
  ↓
[IF confidence > 0.85]
  → [Auto-route based on category]
    - price_inquiry → send price list
    - order_request → create draft order
    - complaint → assign to customer service
  ↓
[IF confidence < 0.85]
  → [Create human_review_task]
  → [Notify support team Telegram]
```

### Pattern 3: Daily report

```
[Schedule: 0 9 * * * (9AM)]
  ↓
[Postgres: aggregate yesterday data]
  - Total orders
  - Total revenue
  - Top products
  - Issues count
  ↓
[Function: format report]
  ↓
[Send to Telegram channel "wecha-daily-report"]
[Send email to leadership]
```

---

## Anti-patterns CẤM

- ❌ Hardcode credentials trong workflow JSON
- ❌ Workflow chứa business logic phức tạp (logic ở ERP API, n8n chỉ orchestrate)
- ❌ Chain webhook không có timeout
- ❌ Loop không có break condition
- ❌ Workflow không log execution
- ❌ Không version control (chỉ ở UI n8n)
- ❌ AI call không có cost tracking
- ❌ External API call không có retry + circuit breaker

---

**END Skill: n8n Workflow**
