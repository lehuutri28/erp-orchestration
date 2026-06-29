---
name: frontend-component
description: Build React/Next component có form, state, data fetching. Hỗ trợ form react-hook-form, TanStack Query, optimistic updates.
when_to_use:
  - Tạo page mới
  - Tạo component có form
  - Component cần fetch data
  - Component có business logic phức tạp
---

# Frontend Component Skill

## 1. Quy trình build component

```
1. Determine type
   ├─ Server Component (default)
   └─ Client Component (cần interactivity)

2. Define props interface
   ├─ Required vs optional
   └─ Default values

3. Define data dependencies
   ├─ Server fetch (RSC)
   └─ Client fetch (TanStack Query)

4. UI states
   ├─ Loading (skeleton)
   ├─ Error (with retry)
   ├─ Empty (with CTA)
   └─ Success (data render)

5. Interactions
   ├─ Forms (react-hook-form + Zod)
   └─ Mutations (TanStack mutation + optimistic)

6. Tests (Vitest + RTL)
   ├─ Render correctly
   ├─ User interactions
   └─ Error states

7. Accessibility (WCAG 2.1 AA)

> 📎 **Cross-ref:** Accessibility checklist WCAG canonical xem [rules/05-frontend.md](../../rules/05-frontend.md#accessibility)
```

## 2. Server Component template

```typescript
// app/orders/page.tsx
import { db } from '@/database';
import { OrdersList } from './orders-list';

export const dynamic = 'force-dynamic';  // No cache

export default async function OrdersPage({
  searchParams,
}: {
  searchParams: { status?: string; q?: string };
}) {
  const orders = await db.query.orders.findMany({
    where: and(
      tenantFilter(orders),
      searchParams.status ? eq(orders.status, searchParams.status) : undefined,
    ),
    limit: 20,
    orderBy: desc(orders.created_at),
  });
  
  return (
    <div>
      <h1 className="text-2xl font-bold mb-4">Đơn hàng</h1>
      <OrdersList initialData={orders} />
    </div>
  );
}
```

## 3. Client Component với form

```typescript
'use client';
import { useForm } from 'react-hook-form';
import { zodResolver } from '@hookform/resolvers/zod';
import { useMutation, useQueryClient } from '@tanstack/react-query';
import { createOrderSchema, type CreateOrderInput } from '@wecha/shared/api/orders';

interface CreateOrderFormProps {
  onSuccess?: (orderId: string) => void;
}

export function CreateOrderForm({ onSuccess }: CreateOrderFormProps) {
  const router = useRouter();
  const queryClient = useQueryClient();
  
  const {
    register,
    handleSubmit,
    formState: { errors, isSubmitting },
    setError,
  } = useForm<CreateOrderInput>({
    resolver: zodResolver(createOrderSchema),
    defaultValues: { items: [], customerId: '' },
  });
  
  const { mutate: createOrder } = useMutation({
    mutationFn: (data: CreateOrderInput) =>
      api.orders.create(data, { idempotencyKey: crypto.randomUUID() }),
    onSuccess: (result) => {
      queryClient.invalidateQueries({ queryKey: ['orders'] });
      toast.success('Tạo đơn thành công');
      onSuccess?.(result.data.id);
      router.push(`/orders/${result.data.id}`);
    },
    onError: (err: ApiError) => {
      if (err.code === 'INSUFFICIENT_STOCK') {
        setError('items', { message: err.detail });
      } else {
        toast.error('Có lỗi xảy ra. Vui lòng thử lại.');
      }
    },
  });
  
  return (
    <form onSubmit={handleSubmit(d => createOrder(d))} className="space-y-4">
      <div>
        <label htmlFor="customer" className="block text-sm font-medium">
          Khách hàng <span className="text-red-500">*</span>
        </label>
        <input
          id="customer"
          {...register('customerId')}
          aria-invalid={!!errors.customerId}
          aria-describedby={errors.customerId ? 'customer-error' : undefined}
          className="mt-1 block w-full rounded border px-3 py-2"
        />
        {errors.customerId && (
          <p id="customer-error" role="alert" className="mt-1 text-sm text-red-600">
            {errors.customerId.message}
          </p>
        )}
      </div>
      
      <button
        type="submit"
        disabled={isSubmitting}
        className="w-full rounded bg-blue-600 py-2 text-white disabled:opacity-50"
      >
        {isSubmitting ? 'Đang tạo...' : 'Tạo đơn'}
      </button>
    </form>
  );
}
```

## 4. Loading/Error/Empty states

```typescript
'use client';
export function OrderList() {
  const { data, isLoading, error } = useQuery({
    queryKey: ['orders'],
    queryFn: () => api.orders.list(),
  });
  
  if (isLoading) return <OrderListSkeleton />;
  
  if (error) {
    return (
      <ErrorState
        title="Không tải được danh sách đơn"
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
        description="Tạo đơn hàng đầu tiên."
        action={<Button onClick={() => router.push('/orders/new')}>Tạo đơn</Button>}
      />
    );
  }
  
  return <div className="space-y-2">{data.map(o => <OrderRow key={o.id} order={o} />)}</div>;
}

function OrderListSkeleton() {
  return (
    <div className="space-y-3">
      {Array.from({ length: 5 }).map((_, i) => (
        <div key={i} className="h-16 bg-gray-200 rounded animate-pulse" />
      ))}
    </div>
  );
}
```

## 5. Optimistic updates

```typescript
const { mutate: cancelOrder } = useMutation({
  mutationFn: (orderId: string) => api.orders.cancel(orderId),
  onMutate: async (orderId) => {
    await queryClient.cancelQueries(['orders', orderId]);
    const previous = queryClient.getQueryData(['orders', orderId]);
    queryClient.setQueryData(['orders', orderId], (old: Order) => ({
      ...old, status: 'cancelled',
    }));
    return { previous };
  },
  onError: (err, orderId, context) => {
    queryClient.setQueryData(['orders', orderId], context.previous);
    toast.error('Huỷ đơn thất bại');
  },
  onSettled: (data, err, orderId) => {
    queryClient.invalidateQueries(['orders', orderId]);
  },
});
```

## 6. Accessibility checklist

- [ ] Mọi button có text hoặc aria-label
- [ ] Mọi input có label liên kết
- [ ] Color contrast ≥ 4.5:1
- [ ] Focus visible
- [ ] Keyboard navigation (Tab, Enter, Esc)
- [ ] Form error có role="alert" và aria-invalid
- [ ] Loading có aria-busy
- [ ] Modal có focus trap + Esc to close
