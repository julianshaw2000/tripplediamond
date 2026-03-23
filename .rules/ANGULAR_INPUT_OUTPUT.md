# Angular – Component API Rules: Inputs, Outputs & Signals (Angular 21+)

---

## 1. Inputs — `input<T>()`

Use `input<T>()` for all component inputs. Never use `@Input()`.

```ts
import { Component, input, computed } from '@angular/core';

@Component({ selector: 'app-user-card', standalone: true, template: `...` })
export class UserCardComponent {
  // Required input — caller must provide a value
  user = input.required<User>();

  // Optional input with default
  compact = input(false);

  // Input with transform — clean at the boundary
  label = input('', {
    transform: (v: string | undefined) => (v ?? '').trim(),
  });

  // Derived from inputs — pure, no side effects
  displayName = computed(() => `${this.user().firstName} ${this.user().lastName}`);
  avatarUrl = computed(() => this.user().avatarUrl ?? '/assets/default-avatar.png');
}
```

**Rules:**
- Always type inputs. `input<User>()` not `input()`.
- Use `input.required<T>()` when a value is always needed — no runtime default check required.
- Use `transform` for trimming, normalising case, parsing — not for validation.
- Inputs are **read-only signals** inside the child. Never call `.set()` on them.
- Never duplicate input state into a local `signal()` — derive with `computed()` instead.

---

## 2. Outputs — `output<T>()`

Use `output<T>()` for all outputs. Never use `@Output()` / `EventEmitter`.

```ts
import { Component, output } from '@angular/core';

@Component({ selector: 'app-order-list', standalone: true, template: `...` })
export class OrderListComponent {
  orderSelected = output<Order>();
  orderDeleted  = output<string>();   // emits order ID
  paginated     = output<PageEvent>();

  onRowClick(order: Order) {
    this.orderSelected.emit(order);
  }
}
```

**Rules:**
- Name outputs with **past tense** (`orderSelected`, `formSubmitted`, `dialogClosed`) or change events (`pageChanged`, `filterChanged`).
- Emit **plain data only** — no services, DOM nodes, or large graphs.
- Emit from explicit user actions or clear state transitions — not from `effect()` or lifecycle hooks.
- Outputs are fire-and-forget; the parent decides what to do with the event.

---

## 3. Two-Way Binding — `model<T>()`

Use `model<T>()` when a component both accepts a value and emits changes back (replaces `@Input` + `@Output` pairs for two-way scenarios).

```ts
import { Component, model } from '@angular/core';

@Component({
  selector: 'app-toggle',
  standalone: true,
  template: `<button (click)="toggle()">{{ enabled() ? 'On' : 'Off' }}</button>`,
})
export class ToggleComponent {
  enabled = model(false);   // creates both input() and output() under the hood

  toggle() {
    this.enabled.set(!this.enabled());
  }
}
```

Parent usage:
```html
<app-toggle [(enabled)]="featureEnabled" />
```

- Use `model()` for form-control-like components, toggle switches, stepper values.
- Do not use `model()` for complex objects that require validation before update — use `input()` + `output()` instead.

---

## 4. Derived State — `computed()`

Any value derived from inputs, injected signals, or local state must live in `computed()`.

```ts
export class ProductListComponent {
  products = input<Product[]>([]);
  filter   = input('');
  sortBy   = input<'name' | 'price'>('name');

  filteredProducts = computed(() => {
    const f = this.filter().toLowerCase();
    return this.products()
      .filter(p => p.name.toLowerCase().includes(f))
      .sort((a, b) => a[this.sortBy()] > b[this.sortBy()] ? 1 : -1);
  });

  totalCount = computed(() => this.filteredProducts().length);
}
```

**Rules:**
- `computed()` functions must be **pure** — no HTTP calls, no state writes, no logging.
- Never recalculate inside the template — always surface via a named `computed()`.
- Do not use `ngOnChanges` where `computed()` achieves the same result.

---

## 5. Parent Owns Writable State

Smart (container) components own `signal()` state. Dumb (presentational) components receive via `input()` and emit intent via `output()`.

```ts
// Smart — owns state, coordinates
@Component({
  selector: 'app-orders',
  standalone: true,
  imports: [OrderListComponent, OrderFiltersComponent],
  template: `
    <app-order-filters
      [value]="filters()"
      (filterChanged)="filters.set($event)"
    />
    <app-order-list
      [orders]="filteredOrders()"
      [isLoading]="facade.isLoading()"
      (orderSelected)="onOrderSelected($event)"
    />
  `,
})
export class OrdersComponent {
  protected facade = inject(OrderFacade);
  filters = signal<OrderFilters>({ status: 'all', search: '' });

  filteredOrders = computed(() =>
    this.facade.orders().filter(o => matchesFilter(o, this.filters()))
  );

  onOrderSelected(order: Order) {
    this.facade.selectOrder(order);
  }
}
```

---

## 6. Signal Queries — `viewChild`, `viewChildren`, `contentChild`

Use signal-based query APIs. Never use `@ViewChild` / `@ContentChild` decorators.

```ts
import { viewChild, viewChildren, contentChild, ElementRef } from '@angular/core';

export class DropdownComponent {
  trigger  = viewChild.required<ElementRef>('trigger');
  items    = viewChildren(DropdownItemComponent);
  label    = contentChild(LabelComponent);

  open() {
    this.trigger().nativeElement.focus();
  }
}
```

---

## 7. `linkedSignal()` — Derived Writable State

Use `linkedSignal()` when you need a writable signal whose default value tracks another signal but can also be overridden locally. (Angular 21+)

```ts
import { linkedSignal } from '@angular/core';

export class PaginatedListComponent {
  pageSize = input(10);

  // Resets to the input default whenever pageSize changes, but is locally overridable
  currentPage = linkedSignal(() => ({ page: 1, size: this.pageSize() }));

  nextPage() {
    this.currentPage.update(p => ({ ...p, page: p.page + 1 }));
  }
}
```

Use `linkedSignal()` for:
- Local UI state that resets when a driving input changes (pagination, selected tab).
- Editable copies of an input that should sync on reset.

Do **not** use it as a substitute for `computed()` when the signal doesn't need to be writable.

---

## 8. Effects — `effect()`

Use `effect()` only for side effects that must react to signal changes. Keep them minimal.

```ts
export class ThemeComponent {
  theme = input<'light' | 'dark'>('light');

  constructor() {
    effect(() => {
      document.body.setAttribute('data-theme', this.theme());
    });
  }
}
```

**Rules:**
- Effects run after render. Do not write to signals inside effects unless using `allowSignalWrites: true` (rare, justified cases only).
- Prefer `computed()` over `effect()` for derived values.
- Never use `effect()` for data fetching — use `resource()` / `httpResource()`.
- Never emit `output()` from inside an `effect()` unless no other design works.

---

## 9. Anti-Patterns

| Anti-pattern | Replace with |
|---|---|
| `@Input()` / `@Output()` decorators | `input()` / `output()` / `model()` |
| `ngOnChanges` for derived values | `computed()` |
| Local signal duplicating an input | `computed()` or `linkedSignal()` |
| Mutating an input inside the child | `output()` to request the change from parent |
| `@ViewChild` / `@ContentChild` decorators | `viewChild()` / `contentChild()` |
| `EventEmitter` | `output<T>()` |
| Emitting complex objects (services, DOM refs) | Emit plain data only |
| `new Subject()` for local state | `signal()` |

---

## 10. Component Checklist

- [ ] Inputs: `input<T>()` / `input.required<T>()`, typed, default provided or required
- [ ] `transform` used for boundary cleanup (trim, normalise, parse)
- [ ] Derived state in `computed()` — pure, no side effects
- [ ] Outputs: `output<T>()`, past-tense name, emits plain data
- [ ] Two-way binding: `model<T>()` where appropriate
- [ ] Smart component owns `signal()` state; dumb component is input/output only
- [ ] `viewChild` / `contentChild` signal queries (no decorators)
- [ ] `ChangeDetectionStrategy.OnPush` on presentational components
- [ ] `effect()` only for side effects, not derived values
