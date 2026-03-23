# Angular – Architecture & Design Pattern Rules (Angular 21+)

Follow these rules exactly when writing or generating Angular code.

---

## 1. Project Structure

```
src/app/
  core/               ← singleton services, interceptors, guards, global error handling, app shell
  shared/             ← feature-agnostic UI components, pipes, directives, utilities
  features/           ← vertical feature slices
    <feature>/
      data/           ← data access services, API adapters, models
      ui/             ← dumb/presentational components for this feature
      <feature>.routes.ts
      <feature>.facade.ts   ← optional: wraps state + data for smart components
  app.config.ts
  app.routes.ts
```

**Dependency rules (hard):**
- `shared/` → Angular only. No `core/` or `features/` imports.
- `core/` → Angular + `shared/`. No `features/` imports.
- `features/` → `core/` + `shared/`. Never cross-import between features.
- Cross-feature shared logic goes into `core/` (services) or `shared/` (UI).

---

## 2. Standalone-First

- All components, directives, pipes: `standalone: true`. No NgModules unless maintaining legacy code.
- Import only what the component needs in its `imports: []` array.
- Lazy-load every feature route via `loadChildren` or `loadComponent` in `app.routes.ts`.
- Provide root-level services with `providedIn: 'root'`. Use `providers: []` on routes for scoped services.

---

## 3. Naming Conventions

- Files: `kebab-case` — `order-list.component.ts`, `user-profile.service.ts`.
- Classes: `PascalCase`. Suffix with type: `OrderListComponent`, `UserService`, `AuthGuard`.
- Signals and computed: noun or adjective — `orders`, `isLoading`, `filteredOrders`.
- Outputs: past tense verb — `orderSelected`, `formSubmitted`, `dialogClosed`.
- Spec files: `*.spec.ts` alongside the file under test.
- Public barrel: `index.ts` per feature for controlled public API surface.

---

## 4. Smart / Dumb Component Pattern

Split every feature into **smart (container)** and **dumb (presentational)** components.

**Smart component** (`features/<feature>/`)
- Holds or injects state (signals, facade, store).
- Calls services, handles routing, dispatches side effects.
- Passes data down via `input()`, receives events via `output()`.
- No inline HTML complexity — delegates to dumb components.

**Dumb component** (`features/<feature>/ui/` or `shared/`)
- Receives all data via `input()`. No direct service injection for data.
- Emits intent via `output()`. Never mutates parent state.
- Fully reusable and testable in isolation.
- Use `ChangeDetectionStrategy.OnPush` on dumb components.

```ts
// smart — features/orders/order-shell.component.ts
@Component({
  selector: 'app-order-shell',
  standalone: true,
  imports: [OrderListComponent],
  template: `
    <app-order-list
      [orders]="facade.orders()"
      [isLoading]="facade.isLoading()"
      (orderSelected)="onOrderSelected($event)"
    />
  `,
})
export class OrderShellComponent {
  protected facade = inject(OrderFacade);

  onOrderSelected(order: Order) {
    this.facade.selectOrder(order);
  }
}
```

---

## 5. Facade Pattern

Use a facade service per feature to decouple smart components from implementation details of state and data access.

```ts
// features/orders/order.facade.ts
@Injectable({ providedIn: 'root' })
export class OrderFacade {
  private store = inject(OrderStore);
  private api = inject(OrderApiService);

  // Expose read-only signals
  readonly orders = this.store.orders.asReadonly();
  readonly selectedOrder = this.store.selectedOrder.asReadonly();
  readonly isLoading = this.store.isLoading.asReadonly();

  loadOrders() { this.store.loadOrders(); }
  selectOrder(order: Order) { this.store.select(order); }
}
```

- Components inject the facade only — never the store or API service directly.
- The facade is the feature's public API surface for smart components.

---

## 6. Signal-Based State (Feature Store Pattern)

Use a lightweight signal store per feature. Avoid NgRx or heavy state libraries unless the app has complex cross-feature state that genuinely requires it.

```ts
// features/orders/order.store.ts
@Injectable({ providedIn: 'root' })
export class OrderStore {
  // Private writable state
  private _orders = signal<Order[]>([]);
  private _selectedOrder = signal<Order | null>(null);
  private _isLoading = signal(false);
  private _error = signal<string | null>(null);

  // Public read-only
  readonly orders = this._orders.asReadonly();
  readonly selectedOrder = this._selectedOrder.asReadonly();
  readonly isLoading = this._isLoading.asReadonly();
  readonly error = this._error.asReadonly();

  // Derived state
  readonly hasOrders = computed(() => this._orders().length > 0);

  private api = inject(OrderApiService);

  loadOrders() {
    this._isLoading.set(true);
    this._error.set(null);
    // use rxResource / httpResource in the API service; update store from effect or callback
  }

  select(order: Order) { this._selectedOrder.set(order); }
}
```

- All writable signals are `private`. Components never call `.set()` on store state directly.
- Derived values are `computed()` on the store, not recalculated per component.
- For cross-feature shared state (user session, notifications), put the store in `core/`.

---

## 7. Adapter Pattern (API → Domain)

Never pass raw API response shapes into components or store. Transform at the data-access boundary.

```ts
// features/orders/data/order.adapter.ts
export function toOrder(dto: OrderDto): Order {
  return {
    id: dto.order_id,
    total: dto.total_amount_cents / 100,
    placedAt: new Date(dto.created_at),
    status: mapOrderStatus(dto.status),
  };
}
```

- One adapter function per domain type.
- Called inside the data access service immediately after the HTTP response arrives.
- Domain model types live in `features/<feature>/data/<feature>.model.ts`.

---

## 8. Functional Providers (guards, resolvers, interceptors)

Prefer functional forms over class-based for all guards, resolvers, and interceptors.

```ts
// core/auth/auth.guard.ts
export const authGuard: CanActivateFn = () => {
  const auth = inject(AuthService);
  const router = inject(Router);
  return auth.isAuthenticated() ? true : router.parseUrl('/login');
};

// core/http/auth.interceptor.ts
export const authInterceptor: HttpInterceptorFn = (req, next) => {
  const token = inject(AuthService).token();
  if (!token) return next(req);
  return next(req.clone({ setHeaders: { Authorization: `Bearer ${token}` } }));
};
```

Register in `app.config.ts`:
```ts
provideHttpClient(withInterceptors([authInterceptor, errorInterceptor])),
provideRouter(routes, withComponentInputBinding()),
```

---

## 9. Template Control Flow & Deferred Loading

Use Angular 17+ built-in control flow (`@if`, `@for`, `@switch`) — no `*ngIf`, `*ngFor`, `*ngSwitch`.

```html
@if (isLoading()) {
  <app-skeleton />
} @else if (error()) {
  <app-error-message [message]="error()" />
} @else {
  @for (order of orders(); track order.id) {
    <app-order-card [order]="order" (selected)="onSelect(order)" />
  } @empty {
    <p>No orders found.</p>
  }
}
```

Use `@defer` for non-critical UI sections to improve initial load performance:

```html
@defer (on viewport) {
  <app-order-analytics [orders]="orders()" />
} @placeholder {
  <div class="analytics-skeleton"></div>
} @loading (minimum 200ms) {
  <app-spinner />
}
```

---

## 10. Routing Best Practices

```ts
// app.routes.ts
export const routes: Routes = [
  {
    path: 'orders',
    loadChildren: () => import('./features/orders/orders.routes').then(m => m.ORDER_ROUTES),
    canActivate: [authGuard],
  },
];

// features/orders/orders.routes.ts
export const ORDER_ROUTES: Routes = [
  { path: '', component: OrderShellComponent },
  { path: ':id', loadComponent: () => import('./order-detail.component').then(m => m.OrderDetailComponent) },
];
```

- Use `withComponentInputBinding()` in `provideRouter` to bind route params/query params directly to `input()`.
- Use functional resolvers to pre-load data before navigation:
  ```ts
  export const orderResolver: ResolveFn<Order> = (route) =>
    inject(OrderApiService).getOrder(route.params['id']);
  ```
- Scope feature-level services to the route with `providers: [OrderStore, OrderFacade]` on the route object.

---

## 11. Change Detection

- Use `ChangeDetectionStrategy.OnPush` on all dumb/presentational components.
- Smart components using signals benefit from fine-grained reactivity automatically — `OnPush` is still recommended.
- Never use `ChangeDetectorRef.markForCheck()` as a workaround — fix the data flow instead.
- Never use `ChangeDetectorRef.detectChanges()` except in very specific cases (e.g. after `@ViewChild` setup in `ngAfterViewInit`).

---

## 12. RxJS Interop

Use `toSignal()` and `toObservable()` to cross the boundary between RxJS streams and signals. Avoid keeping both forms of the same state alive simultaneously.

```ts
// Convert Observable → Signal (preferred for component consumption)
readonly user = toSignal(this.userService.user$, { initialValue: null });

// Convert Signal → Observable (for operators like debounce, switchMap)
readonly search$ = toObservable(this.searchQuery);
readonly results$ = this.search$.pipe(
  debounceTime(300),
  switchMap(q => this.api.search(q)),
);
readonly results = toSignal(this.results$, { initialValue: [] });
```

- Always provide `{ initialValue }` to `toSignal()` to avoid `undefined` in templates.
- Use `takeUntilDestroyed()` for any remaining manual subscriptions — never unsubscribe in `ngOnDestroy`.

---

## 13. Code Output Rules

When generating code:
- Provide the file path at the top of each snippet.
- Keep feature code in `src/app/features/<feature>/`.
- Keep global singletons in `src/app/core/`.
- Keep reusable UI/utilities in `src/app/shared/`.
- Export feature public API only through the feature's `index.ts`.
- Changes must be small and consistent with existing project conventions.

---

## 14. Non-Negotiables

- No NgModules in new code.
- No `constructor()` DI — use `inject()` everywhere.
- No `async` pipe — use signals and `toSignal()`.
- No `*ngIf` / `*ngFor` — use `@if` / `@for`.
- No `@Input()` / `@Output()` decorators — use `input()` / `output()` / `model()`.
- No `Subject` for state management — use `signal()`.
- No direct cross-feature imports.
- Do not duplicate domain logic across features.
