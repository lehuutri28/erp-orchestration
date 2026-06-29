# Rule 05 — Frontend (Next.js 14 + React 18)

> Mức độ: 🟠 HIGH | Áp dụng: Tạo page, component, form, state

---

## 1. Server vs Client Components

**Mặc định Server Component, chỉ dùng Client khi cần.**

```typescript
// ✅ Server Component (default) — fetch data, render HTML
export default async function OrdersPage() {
  const orders = await fetchOrdersServerSide();  // Direct DB or API call
  return <OrderList orders={orders} />;
}

// ✅ Client Component (chỉ khi cần interactivity)
'use client';

export function OrderFilterBar({ onFilter }: Props) {
  const [status, setStatus] = useState<string>('all');
  return <select onChange={(e) => { setStatus(e.target.value); onFilter(e.target.value); }} />;
}
```

**Khi nào dùng Client Component:**
- Cần `useState`, `useEffect`, `useReducer`
- Cần event handlers (`onClick`, `onChange`, `onSubmit`)
- Cần browser API (`localStorage`, `window`, `document`)
- Cần lifecycle hooks
- Dùng custom hook khác

**KHÔNG dùng Client cho:**
- Chỉ render data tĩnh
- Layout/wrapper không có logic

---

## 2. Data fetching pattern

### Server-side (RSC)
```typescript
// app/orders/page.tsx
import { db } from '@/database';
import { OrdersList } from './orders-list';

export default async function OrdersPage({
  searchParams,
}: {
  searchParams: { status?: string; cursor?: string };
}) {
  // Direct DB query (Server Component)
  const orders = await db.query.orders.findMany({
    where: and(
      tenantFilter(orders),
      searchParams.status ? eq(orders.status, searchParams.status) : undefined,
    ),
    limit: 21,
  });
  
  return <OrdersList initialData={orders} />;
}
```

### Client-side (TanStack Query) — Wave 4
```typescript
'use client';
import { useInfiniteQuery } from '@tanstack/react-query';

export function OrdersList({ initialData }: { initialData: Order[] }) {
  const { data, fetchNextPage, hasNextPage, isLoading } = useInfiniteQuery({

> 📎 **Cross-ref:** Code đầy đủ 3-state pattern (Loading/Error/Empty) xem [skills/frontend-component/SKILL.md](../skills/frontend-component/SKILL.md#3-state-pattern)
    queryKey: ['orders', { status: 'all' }],
    queryFn: ({ pageParam }) => fetchOrders({ cursor: pageParam }),
    initialPageParam: undefined,
    getNextPageParam: (lastPage) => lastPage.meta.nextCursor,
    initialData: { pages: [{ data: initialData }], pageParams: [undefined] },
    staleTime: 60 * 1000,        // 1 phút
    gcTime: 5 * 60 * 1000,       // 5 phút
  });
  
  if (isLoading) return <OrderListSkeleton />;
  
  return (
    <div>
      {data.pages.flatMap(p => p.data).map(order => <OrderRow key={order.id} order={order} />)}
      {hasNextPage && <LoadMoreButton onClick={() => fetchNextPage()} />}
    </div>
  );
}
```

---

## 3. State management — quy tắc chọn tool

| Loại state | Tool | Khi nào |
|---|---|---|
| Server state (data từ API) | **TanStack Query** | Mọi data fetch |
| URL state (filter, page, modal mở) | **URL search params** | State cần share qua link |
| Form state | **react-hook-form** | Mọi form > 3 fields |
| Component local state | **useState** | UI state đơn giản (toggle, hover) |
| Cross-component (cùng route) | **React Context** | < 5 component subscribe |
| Global app state | **Zustand** | Theme, sidebar collapsed, notification queue |

**🔴 CẤM:** Redux (over-engineered cho project này), Recoil (deprecated), MobX.

---

## 4. Form pattern — react-hook-form + Zod

```typescript
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { createOrderSchema, type CreateOrderInput } from '@wecha/shared/api/orders';

export function CreateOrderForm() {
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setError,
  } = useForm<CreateOrderInput>({
    resolver: zodResolver(createOrderSchema),
    defaultValues: { items: [], customerId: '' },
  });
  
  const onSubmit = async (data: CreateOrderInput) => {
    try {
      const result = await api.orders.create(data, {
        idempotencyKey: crypto.randomUUID(),
      });
      router.push(`/orders/${result.data.id}`);
      toast.success('Tạo đơn thành công');
    } catch (err) {
      if (err instanceof ApiError && err.code === 'INSUFFICIENT_STOCK') {
        setError('items', { message: err.detail });
      } else {
        toast.error('Có lỗi xảy ra. Vui lòng thử lại.');
      }
    }
  };
  
  return (
    <form onSubmit={handleSubmit(onSubmit)}>
      <label>
        <span>Khách hàng</span>
        <input {...register('customerId')} aria-invalid={!!errors.customerId} />
        {errors.customerId && <span role="alert">{errors.customerId.message}</span>}
      </label>
      
      {/* ... */}
      
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? 'Đang tạo...' : 'Tạo đơn'}
      </button>
    </form>
  );
}
```

**Validation schema chia sẻ FE+BE:**
```typescript
// packages/shared/src/api/orders/schemas.ts
import { z } from 'zod';

export const createOrderSchema = z.object({
  customerId: z.string().min(1, 'Khách hàng bắt buộc'),
  items: z.array(z.object({
    productId: z.string(),
    quantity: z.number().int().positive('Số lượng phải > 0'),
  })).min(1, 'Phải có ít nhất 1 sản phẩm'),
  customerNote: z.string().max(500).optional(),
}).strict();

export type CreateOrderInput = z.infer<typeof createOrderSchema>;
```

→ FE dùng cho form validation, BE dùng cho request validation. Đổi 1 nơi → 2 nơi auto sync.

---

## 5. Component structure

```
features/orders/
├── components/
│   ├── OrderList.tsx
│   ├── OrderRow.tsx
│   ├── OrderForm.tsx
│   ├── OrderStatusBadge.tsx
│   └── index.ts
├── hooks/
│   ├── use-orders.ts            # TanStack Query wrapper
│   ├── use-create-order.ts
│   └── use-cancel-order.ts
├── api/
│   └── orders.api.ts            # Fetch functions
├── types/
│   └── order.types.ts
└── utils/
    └── format-order.ts
```

**Pattern component:**
```typescript
// 1. Imports order: external → internal → relative → types
import { useState } from 'react';
import { useRouter } from 'next/navigation';
import { Button } from '@wecha/ui';
import { formatVnd } from '@wecha/shared/utils';
import { useCancelOrder } from '../hooks/use-cancel-order';
import type { Order } from '../types/order.types';

// 2. Props interface (KHÔNG type alias)
interface OrderRowProps {
  order: Order;
  onUpdate?: (order: Order) => void;
  className?: string;
}

// 3. Component
export function OrderRow({ order, onUpdate, className }: OrderRowProps) {
  const router = useRouter();
  const { mutate: cancel, isPending } = useCancelOrder();
  
  return (
    <div className={cn('flex items-center justify-between p-4', className)}>
      <div>
        <h3 className="font-medium">{order.code}</h3>
        <p className="text-sm text-gray-500">{order.customer_snapshot.name}</p>
      </div>
      
      <div className="flex items-center gap-2">
        <span>{formatVnd(order.total)}</span>
        <Button variant="ghost" onClick={() => router.push(`/orders/${order.id}`)}>
          Xem
        </Button>
      </div>
    </div>
  );
}
```

---

## 6. Loading + Error + Empty states

**MỌI data display PHẢI có 3 state:**

```typescript
'use client';

export function OrderList() {
  const { data, isLoading, error } = useOrders();
  
  if (isLoading) return <OrderListSkeleton />;
  
  if (error) {
    return (
      <ErrorState
        title="Không tải được danh sách đơn hàng"
        message={error.message}
        onRetry={() => queryClient.invalidateQueries(['orders'])}
      />
    );
  }
  
  if (data.length === 0) {
    return (
      <EmptyState
        icon={<ShoppingBag />}
        title="Chưa có đơn hàng nào"
        description="Tạo đơn hàng đầu tiên để bắt đầu."
        action={<Button onClick={() => router.push('/orders/new')}>Tạo đơn</Button>}
      />
    );
  }
  
  return <div>{data.map(o => <OrderRow key={o.id} order={o} />)}</div>;
}
```

**Skeleton placeholder** (BẮT BUỘC cho list):
```typescript
export function OrderListSkeleton() {
  return (
    <div className="space-y-3">
      {Array.from({ length: 5 }).map((_, i) => (
        <div key={i} className="flex items-center justify-between p-4 animate-pulse">
          <div className="space-y-2">
            <div className="h-4 w-32 bg-gray-200 rounded" />
            <div className="h-3 w-48 bg-gray-200 rounded" />
          </div>
          <div className="h-8 w-24 bg-gray-200 rounded" />
        </div>
      ))}
    </div>
  );
}
```

---

## 7. Optimistic updates

> 📎 **Cross-ref:** Code đầy đủ Optimistic updates pattern xem [skills/frontend-component/SKILL.md](../skills/frontend-component/SKILL.md#optimistic-updates)

```typescript
const { mutate: cancelOrder } = useMutation({
  mutationFn: (orderId: string) => api.orders.cancel(orderId),
  onMutate: async (orderId) => {
    await queryClient.cancelQueries(['orders', orderId]);
    const previous = queryClient.getQueryData(['orders', orderId]);
    
    queryClient.setQueryData(['orders', orderId], (old: Order) => ({
      ...old,
      status: 'cancelled',
    }));
    
    return { previous };  // Snapshot để rollback
  },
  onError: (err, orderId, context) => {
    queryClient.setQueryData(['orders', orderId], context.previous);  // Rollback
    toast.error('Huỷ đơn thất bại');
  },
  onSettled: (data, err, orderId) => {
    queryClient.invalidateQueries(['orders', orderId]);
  },
});
```

---

## 8. i18n (next-intl) — Wave 5

**KHÔNG hardcode text:**
```typescript
// ❌ SAI
<button>Tạo đơn</button>

// ✅ ĐÚNG
import { useTranslations } from 'next-intl';
const t = useTranslations('Orders');
<button>{t('create')}</button>
```

```json
// messages/vi.json
{ "Orders": { "create": "Tạo đơn", "cancel": "Huỷ đơn" } }

// messages/en.json
{ "Orders": { "create": "Create order", "cancel": "Cancel order" } }
```

---

## 9. Image optimization

**LUÔN dùng `next/image`:**
```typescript
// ❌ SAI
<img src="/product.jpg" alt="Product" />

// ✅ ĐÚNG
import Image from 'next/image';

<Image
  src="/product.jpg"
  alt="Trà sữa size M"
  width={200}
  height={200}
  loading="lazy"          // Default (trừ above-fold)
  placeholder="blur"
  blurDataURL={base64Blur}
/>

// Cho ảnh remote
<Image
  src={`https://r2.wecha.vn/${product.image}`}
  alt={product.name}
  width={200}
  height={200}
  unoptimized={false}     // Default — Next.js optimize
/>
```

`next.config.mjs`:
```js
images: {
  remotePatterns: [
    { protocol: 'https', hostname: 'r2.wecha.vn' },
    { protocol: 'https', hostname: 'cdn.wecha.vn' },
  ],
  formats: ['image/avif', 'image/webp'],
}
```

---

## 10. Performance

**Lazy load component nặng:**
```typescript
import dynamic from 'next/dynamic';

const HeavyChart = dynamic(() => import('./heavy-chart'), {
  loading: () => <ChartSkeleton />,
  ssr: false,  // Nếu chỉ chạy client
});
```

**Tránh re-render thừa:**
```typescript
// useMemo cho computed value đắt
const total = useMemo(() => calculateTotal(items), [items]);

// useCallback cho function pass xuống
const handleClick = useCallback((id: string) => { /* ... */ }, [dependency]);

// memo cho component nhận props
export const OrderRow = memo(function OrderRow({ order }: Props) { /* ... */ });
```

**Bundle analysis:**
```bash
ANALYZE=true pnpm build
# → mở reports/client.html
```

---

## 11. Accessibility (WCAG 2.1 AA)

**Checklist:**
- [ ] Mọi button có text hoặc `aria-label`
- [ ] Mọi input có `<label>` liên kết
- [ ] Mọi image có `alt` (rỗng `""` nếu decorative)
- [ ] Color contrast ≥ 4.5:1 cho text thường, 3:1 cho text lớn
- [ ] Focus visible (không `outline: none` mà không có thay thế)
- [ ] Keyboard navigation hoạt động (Tab, Enter, Esc)
- [ ] Form error có `role="alert"` và `aria-invalid`
- [ ] Loading state có `aria-busy` và screen reader text
- [ ] Modal có focus trap + Esc to close
- [ ] Heading hierarchy đúng (h1 → h2 → h3, không skip)

```typescript
// Modal accessible
<Dialog open={isOpen} onOpenChange={setIsOpen}>
  <DialogContent
    aria-labelledby="dialog-title"
    aria-describedby="dialog-desc"
  >
    <DialogTitle id="dialog-title">Xác nhận huỷ đơn</DialogTitle>
    <p id="dialog-desc">Đơn hàng sẽ bị huỷ vĩnh viễn. Tiếp tục?</p>
    <DialogClose>Đóng</DialogClose>
  </DialogContent>
</Dialog>
```

---

## 12. SEO + meta tags (Next.js Metadata API)

```typescript
// app/orders/page.tsx
import type { Metadata } from 'next';

export const metadata: Metadata = {
  title: 'Đơn hàng | WECHA ERP',
  description: 'Quản lý đơn hàng — WECHA chuỗi trà sữa nhượng quyền',
  openGraph: {
    title: 'Đơn hàng | WECHA ERP',
    description: '...',
    images: ['/og-orders.jpg'],
  },
  robots: { index: false, follow: false },  // Internal admin
};
```

---

## 13. Cấm tuyệt đối 🔴

- `ignoreBuildErrors: true` trong `next.config.mjs`
- `ignoreDuringBuilds: true` cho ESLint
- `// @ts-nocheck` ở đầu file
- `<img>` thay `next/image`
- `dangerouslySetInnerHTML` không qua DOMPurify
- localStorage cho data nhạy cảm
- Inline `<style>` lớn (dùng Tailwind hoặc CSS module)
- `console.log` trong production code

---

## 14. Cấu trúc tsconfig.json BẮT BUỘC 🔴

```json
{
  "compilerOptions": {
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noImplicitReturns": true,
    "noFallthroughCasesInSwitch": true,
    "noUnusedLocals": true,
    "noUnusedParameters": true,
    "exactOptionalPropertyTypes": true,
    "isolatedModules": true,
    "verbatimModuleSyntax": true
  }
}
```

---

**END Rule 05**
