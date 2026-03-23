# Angular – HTTP & Data Access Rules (Angular 21+)

---

## 1. Where HTTP Lives

All HTTP calls belong in **data access services** inside `features/<feature>/data/` or `core/`.
Components and facades never touch `HttpClient` or `resource()` directly.

```
features/orders/
  data/
    order-api.service.ts    ← httpResource / HttpClient calls + adapter mapping
    order.model.ts          ← domain interface + DTO interface
    order.adapter.ts        ← DTO → domain transform functions
  order.store.ts            ← consumes the API service, holds signal state
  order.facade.ts           ← exposes store signals + action methods to components
```

---

## 2. Resource API (preferred for declarative data fetching)

Use `httpResource()` for GET requests that are driven by reactive parameters. Use `resource()` when you need full control over the async loader (e.g., POST-based searches).

### `httpResource` — simple GET

```ts
// features/orders/data/order-api.service.ts
@Injectable({ providedIn: 'root' })
export class OrderApiService {
  private readonly baseUrl = inject(API_URL_TOKEN);

  // Reactive: re-fetches whenever orderId signal changes
  orderResource(orderId: Signal<string>) {
    return httpResource<OrderDto>(
      () => `${this.baseUrl}/orders/${orderId()}`,
      { parse: (dto) => toOrder(dto) }   // adapter applied at the boundary
    );
  }

  ordersResource(filters: Signal<OrderFilters>) {
    return httpResource<OrderDto[]>(
      () => ({
        url: `${this.baseUrl}/orders`,
        params: toHttpParams(filters()),
      }),
      { parse: (dtos) => dtos.map(toOrder) }
    );
  }
}
```

### `resource()` — async loader with full control

```ts
searchResource(query: Signal<string>) {
  return resource({
    request: query,
    loader: async ({ request: q, abortSignal }) => {
      const res = await fetch(`${this.baseUrl}/orders/search?q=${q}`, { signal: abortSignal });
      if (!res.ok) throw new Error(`Search failed: ${res.status}`);
      const dtos: OrderDto[] = await res.json();
      return dtos.map(toOrder);
    },
  });
}
```

### `rxResource()` — when the loader needs RxJS operators

```ts
typeaheadResource(query: Signal<string>) {
  return rxResource({
    request: query,
    loader: ({ request: q }) =>
      this.http.get<OrderDto[]>(`${this.baseUrl}/orders`, { params: { q } }).pipe(
        map(dtos => dtos.map(toOrder)),
        catchError(() => of([]))
      ),
  });
}
```

---

## 3. Mutations (POST / PUT / PATCH / DELETE)

Resources are for reads. For mutations use `HttpClient` directly in the data access service and return an `Observable`. Call from the store or facade using `toSignal()` or manage via an `effect()`.

```ts
// In OrderApiService
createOrder(command: CreateOrderCommand): Observable<Order> {
  return this.http
    .post<OrderDto>(`${this.baseUrl}/orders`, command)
    .pipe(map(toOrder));
}

// In OrderStore
create(command: CreateOrderCommand) {
  this._isLoading.set(true);
  inject(OrderApiService).createOrder(command)
    .pipe(takeUntilDestroyed(this.destroyRef))
    .subscribe({
      next: (order) => {
        this._orders.update(list => [...list, order]);
        this._isLoading.set(false);
      },
      error: (err) => {
        this._error.set(extractErrorMessage(err));
        this._isLoading.set(false);
      },
    });
}
```

---

## 4. Expose State to Components — Required Shape

Every resource or store slice exposed to components must provide all four states:

```ts
// In facade or smart component
readonly isLoading = this.store.isLoading;                        // Signal<boolean>
readonly orders    = this.store.orders;                           // Signal<Order[]>
readonly error     = this.store.error;                            // Signal<string | null>
readonly isEmpty   = computed(() => !this.store.isLoading() && this.store.orders().length === 0);
```

For inline resource use (in the store/service layer):
```ts
private resource = this.apiService.ordersResource(this.filters);

readonly isLoading = this.resource.isLoading;
readonly orders    = computed(() => this.resource.value() ?? []);
readonly error     = computed(() => this.resource.error() ?? null);
```

---

## 5. Functional Interceptors

All interceptors are functional. Register once in `app.config.ts`.

### Auth token

```ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  if (!token) return next(req);
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};
```

### Global error handling

```ts
export const errorInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    catchError((err: HttpErrorResponse) => {
      inject(ErrorNotificationService).notify(toErrorMessage(err));
      return throwError(() => err);
    })
  );
```

### Retry with backoff (resilience)

```ts
export const retryInterceptor: HttpInterceptorFn = (req, next) =>
  next(req).pipe(
    retry({ count: 2, delay: (_, attempt) => timer(attempt * 500) })
  );
```

Register:
```ts
// app.config.ts
provideHttpClient(withInterceptors([authInterceptor, errorInterceptor, retryInterceptor]))
```

---

## 6. Caching

For reads that rarely change, implement a simple signal-based cache in the data access service:

```ts
@Injectable({ providedIn: 'root' })
export class ReferenceDataService {
  private _cache = signal<Map<string, ReferenceItem[]>>(new Map());

  get(type: string): Observable<ReferenceItem[]> {
    const cached = this._cache().get(type);
    if (cached) return of(cached);
    return this.http.get<ReferenceItem[]>(`/api/ref/${type}`).pipe(
      tap(items => this._cache.update(m => new Map(m).set(type, items)))
    );
  }

  invalidate(type: string) {
    this._cache.update(m => { const n = new Map(m); n.delete(type); return n; });
  }
}
```

---

## 7. Error Handling Utilities

Centralise error message extraction:

```ts
// core/utils/error.utils.ts
export function extractErrorMessage(err: unknown): string {
  if (err instanceof HttpErrorResponse) {
    return err.error?.detail ?? err.error?.message ?? err.message ?? 'Unknown server error';
  }
  if (err instanceof Error) return err.message;
  return 'An unexpected error occurred';
}
```

---

## 8. Environment Configuration

Use injection tokens for base URLs — never hardcode API paths:

```ts
// core/tokens/api-url.token.ts
export const API_URL_TOKEN = new InjectionToken<string>('API_URL', {
  providedIn: 'root',
  factory: () => inject(ENVIRONMENT).apiUrl,
});
```

---

## 9. Anti-Patterns (Never Do)

| Anti-pattern | Replace with |
|---|---|
| `HttpClient` in a component | Data access service / facade |
| `.subscribe()` in a component for data flow | `toSignal()` or resource signals |
| `async` pipe in templates | Signal returned from `toSignal()` or `resource` |
| `new Subject()` to hold API results | `signal()` in the store |
| Catching errors per-component | Global `errorInterceptor` + store `error` signal |
| Returning `any` from HTTP calls | Typed `HttpClient` + DTO interfaces |
| Calling `resource().reload()` on every keystroke without debounce | Use `rxResource()` with `debounceTime` |
