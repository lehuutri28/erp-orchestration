# Rule 10 — Error Handling

> Mức độ: 🔴 CRITICAL | Áp dụng: Mọi nơi có error

---

## 1. Phân loại error

```typescript
// shared/errors/base.error.ts
export abstract class AppError extends Error {
  abstract readonly code: string;
  abstract readonly status: number;
  abstract readonly isOperational: boolean;  // Predicted vs Programmer error
  
  constructor(
    message: string,
    public readonly context?: Record<string, unknown>,
    public readonly cause?: Error,
  ) {
    super(message);
    this.name = this.constructor.name;
    Error.captureStackTrace(this, this.constructor);
  }
  
  toJSON() {
    return {
      code: this.code,
      message: this.message,
      context: this.context,
    };
  }
}

// Operational errors (expected, không alert)
export class ValidationError extends AppError {
  readonly code = 'VALIDATION_ERROR';
  readonly status = 422;
  readonly isOperational = true;
}

export class NotFoundError extends AppError {
  readonly code = 'NOT_FOUND';
  readonly status = 404;
  readonly isOperational = true;
}

export class UnauthorizedError extends AppError {
  readonly code = 'UNAUTHORIZED';
  readonly status = 401;
  readonly isOperational = true;
}

export class ForbiddenError extends AppError {
  readonly code = 'FORBIDDEN';
  readonly status = 403;
  readonly isOperational = true;
}

export class ConflictError extends AppError {
  readonly code = 'CONFLICT';
  readonly status = 409;
  readonly isOperational = true;
}

// Domain-specific errors
export class InsufficientStockError extends AppError {
  readonly code = 'INSUFFICIENT_STOCK';
  readonly status = 422;
  readonly isOperational = true;
  
  constructor(productId: string, requested: number, available: number) {
    super(`Sản phẩm ${productId} chỉ còn ${available}, không đủ ${requested}`, {
      productId, requested, available,
    });
  }
}

export class PaymentFailedError extends AppError {
  readonly code = 'PAYMENT_FAILED';
  readonly status = 402;
  readonly isOperational = true;
}

// Programmer errors (alert ngay)
export class InternalError extends AppError {
  readonly code = 'INTERNAL_ERROR';
  readonly status = 500;
  readonly isOperational = false;
}

export class ConfigurationError extends AppError {
  readonly code = 'CONFIGURATION_ERROR';
  readonly status = 500;
  readonly isOperational = false;
}
```

---

## 2. Global Exception Filter

```typescript
// shared/filters/global-exception.filter.ts
import { ExceptionFilter, Catch, ArgumentsHost, HttpStatus, Logger } from '@nestjs/common';
import * as Sentry from '@sentry/node';

@Catch()
export class GlobalExceptionFilter implements ExceptionFilter {
  private readonly logger = new Logger('GlobalExceptionFilter');
  
  catch(exception: unknown, host: ArgumentsHost) {
    const ctx = host.switchToHttp();
    const request = ctx.getRequest();
    const response = ctx.getResponse();
    const requestId = request.id ?? randomUUID();
    
    let status: number;
    let body: any;
    
    if (exception instanceof AppError) {
      status = exception.status;
      body = {
        error: {
          code: exception.code,
          title: this.titleFor(exception.code),
          status,
          detail: exception.message,
          context: exception.context,
          instance: request.url,
          requestId,
        }
      };
      
      if (!exception.isOperational) {
        Sentry.captureException(exception, {
          tags: { code: exception.code },
          extra: { requestId, url: request.url },
        });
      }
    }
    else if (exception instanceof HttpException) {
      status = exception.getStatus();
      const httpResponse = exception.getResponse();
      body = {
        error: {
          code: this.mapNestErrorToCode(exception),
          title: typeof httpResponse === 'string' ? httpResponse : (httpResponse as any).message,
          status,
          detail: typeof httpResponse === 'string' ? httpResponse : JSON.stringify(httpResponse),
          instance: request.url,
          requestId,
        }
      };
    }
    else if (exception instanceof ZodError) {
      status = 422;
      body = {
        error: {
          code: 'VALIDATION_ERROR',
          title: 'Dữ liệu không hợp lệ',
          status,
          fields: exception.flatten().fieldErrors,
          requestId,
        }
      };
    }
    else {
      // Unknown error — programmer error
      status = 500;
      body = {
        error: {
          code: 'INTERNAL_ERROR',
          title: 'Đã có lỗi xảy ra',
          status,
          detail: process.env.NODE_ENV === 'production' 
            ? 'Vui lòng thử lại sau' 
            : (exception as Error).message,
          requestId,
        }
      };
      
      this.logger.error(
        `Unhandled exception: ${(exception as Error).message}`,
        (exception as Error).stack,
      );
      
      Sentry.captureException(exception, {
        tags: { url: request.url, method: request.method },
        extra: { requestId, body: request.body },
      });
    }
    
    // Log structured
    this.logger.warn({
      requestId,
      method: request.method,
      url: request.url,
      status,
      errorCode: body.error.code,
      tenantId: request.user?.tenantId,
      userId: request.user?.id,
    });
    
    response.status(status).json(body);
  }
  
  private titleFor(code: string): string {
    const map = {
      'VALIDATION_ERROR': 'Dữ liệu không hợp lệ',
      'NOT_FOUND': 'Không tìm thấy',
      'UNAUTHORIZED': 'Chưa đăng nhập',
      'FORBIDDEN': 'Không có quyền',
      'CONFLICT': 'Xung đột dữ liệu',
      'INSUFFICIENT_STOCK': 'Sản phẩm hết hàng',
      'PAYMENT_FAILED': 'Thanh toán thất bại',
      'RATE_LIMIT_EXCEEDED': 'Quá nhiều yêu cầu',
    };
    return map[code] ?? 'Có lỗi xảy ra';
  }
}
```

---

## 3. Anti-patterns CẤM 🔴

```typescript
// ❌ Nuốt lỗi
try {
  await chargePayment();
} catch (e) {
  // không log gì
}

// ❌ Re-throw mất context
try {
  await chargePayment();
} catch (e) {
  throw new Error('Payment failed');  // Mất stack trace, mất chi tiết
}

// ❌ Throw string
throw 'Payment failed';  // KHÔNG có stack trace

// ❌ Catch tất cả mà không phân loại
try { /* ... */ } catch (e) { return null; }

// ✅ Đúng: log + re-throw với cause
try {
  await chargePayment();
} catch (e) {
  this.logger.error('Charge payment failed', { 
    error: e.message, 
    customerId, 
    amount,
    stack: e.stack,
  });
  throw new PaymentFailedError('Không thể thanh toán', { customerId, amount }, e);
}

// ✅ Đúng: handle specific
try {
  await chargePayment();
} catch (e) {
  if (e instanceof InsufficientFundsError) {
    return { success: false, reason: 'insufficient_funds' };
  }
  if (e instanceof NetworkError) {
    // Retry với backoff
    return await this.retryCharge();
  }
  throw e;  // Unknown — bubble up
}
```

---

## 4. Retry strategy

```typescript
import pRetry from 'p-retry';

async function callExternalApi(payload: any) {
  return pRetry(
    async () => {
      const response = await fetch('https://api.ghn.vn/...', { /* ... */ });
      
      if (response.status >= 500) {
        // 5xx → retry
        throw new Error(`Server error ${response.status}`);
      }
      
      if (response.status >= 400) {
        // 4xx → KHÔNG retry (client error)
        throw new pRetry.AbortError(`Client error ${response.status}`);
      }
      
      return response.json();
    },
    {
      retries: 3,
      factor: 2,           // Exponential
      minTimeout: 1000,    // 1s
      maxTimeout: 30000,   // 30s
      randomize: true,     // Jitter
      onFailedAttempt: (error) => {
        logger.warn(`Retry attempt ${error.attemptNumber} failed: ${error.message}`);
      },
    },
  );
}
```

**Retry rules:**
- ✅ Retry: 5xx, 408, 429, network errors, timeout
- ❌ KHÔNG retry: 4xx (trừ 408, 429), validation errors, auth errors
- Idempotent operations only
- Max 3 attempts (default)
- Exponential backoff với jitter
- Circuit breaker cho repeated failures

---

## 5. Circuit Breaker

```typescript
import CircuitBreaker from 'opossum';

const ghnBreaker = new CircuitBreaker(
  async (orderData) => fetch('https://api.ghn.vn/v2/order/create', { /* ... */ }),
  {
    timeout: 10_000,
    errorThresholdPercentage: 50,    // Mở mạch nếu > 50% fail
    resetTimeout: 5 * 60_000,        // Thử lại sau 5 phút
    rollingCountTimeout: 60_000,     // Window 60s
    rollingCountBuckets: 10,
    name: 'ghn-create-order',
  }
);

ghnBreaker.on('open', () => {
  logger.error('GHN circuit breaker OPEN');
  alerts.send('CIRCUIT_OPEN', { service: 'ghn' });
});

ghnBreaker.on('halfOpen', () => {
  logger.warn('GHN circuit breaker HALF-OPEN — testing');
});

ghnBreaker.fallback(() => ({
  success: false,
  fallback: true,
  message: 'GHN tạm thời không khả dụng. Đơn sẽ được xử lý sau.',
}));

// Usage
const result = await ghnBreaker.fire(orderData);
```

---

**END Rule 10**
