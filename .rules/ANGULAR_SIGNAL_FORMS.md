# Angular – Signal Forms Rules (@angular/forms/signals, Angular 21+)

---

## 1. Core Pattern

Signal Forms use one `signal<TModel>` as the single source of truth. Build the field tree with `form()`, bind inputs with `[formField]`, read state with signal calls.

```ts
import { form, FormField } from '@angular/forms/signals';
import { signal } from '@angular/core';

@Component({
  selector: 'app-customer-form',
  standalone: true,
  imports: [FormField],
  templateUrl: './customer-form.component.html',
})
export class CustomerFormComponent {
  model = signal<Customer>(initialCustomer);
  theForm = form(this.model, customerRules);

  onSubmit(event: SubmitEvent) {
    event.preventDefault();
    if (!this.theForm().valid()) return;
    // pass this.model() to service
  }
}
```

---

## 2. File Structure

```
features/<feature>/
  models/
    <thing>.model.ts        ← interface + initialValue + rules function
  <thing>-form.component.ts
  <thing>-form.component.html
```

### `models/customer.model.ts`

```ts
import { email, min, max, minLength, required, SchemaPath } from '@angular/forms/signals';

export interface Customer {
  name: string;
  email: string;
  useEmail: boolean;
  age: number | null;
}

export const initialCustomer: Customer = {
  name: '',
  email: '',
  useEmail: true,
  age: null,
};

export function customerRules(p: SchemaPath<Customer>) {
  required(p.name, { message: 'Name is required' });
  minLength(p.name, 3, { message: 'Name must be at least 3 characters' });

  required(p.email, {
    message: 'Email is required',
    when: ({ valueOf }) => valueOf(p.useEmail),
  });
  email(p.email, { message: 'Must be a valid email' });

  min(p.age, 13, { message: 'Must be at least 13', when: ({ value }) => value() != null });
  max(p.age, 120, { message: 'Must be 120 or under', when: ({ value }) => value() != null });
}
```

---

## 3. Hard Rules

1. **One `signal<TModel>`** is the single source of truth — no parallel state.
2. **`form(model, rules)`** builds the field tree — do not build it manually.
3. **Validation rules in a separate model file** — never inline in the component.
4. **`[formField]` only** — no `[value]`/`(input)` pairs, no `FormGroup`, no `FormControl`, no `[(ngModel)]`.
5. **Read state via signal calls**: `theForm().name().valid()`, `theForm().name().errors()`, `theForm().name().touched()`.
6. **Submit must call `event.preventDefault()`** — bind with `(submit)="onSubmit($event)"`.
7. **No `ReactiveFormsModule`** imports unless maintaining legacy code.
8. **Do not duplicate model state** in extra signals — use `computed()` for derived UI state only.

---

## 4. Template Pattern

```html
<form (submit)="onSubmit($event)">

  <label>
    Name
    <input [formField]="theForm().name" />
    @if (theForm().name().touched()) {
      @for (e of theForm().name().errors(); track e) {
        <span class="error">{{ e }}</span>
      }
    }
  </label>

  <label>
    <input type="checkbox" [formField]="theForm().useEmail" />
    Receive email updates
  </label>

  @if (theForm().useEmail().value()) {
    <label>
      Email
      <input type="email" [formField]="theForm().email" />
      @if (theForm().email().touched()) {
        @for (e of theForm().email().errors(); track e) {
          <span class="error">{{ e }}</span>
        }
      }
    </label>
  }

  <label>
    Age
    <input type="number" [formField]="theForm().age" />
  </label>

  <button type="submit" [disabled]="!theForm().valid()">Save</button>

</form>
```

---

## 5. Model & Validation Rules

- Use `string: ''`, `boolean: false`, `number | null: null` as initial values — avoid `undefined` or `NaN`.
- Put all validation messages in the rules function — never in templates.
- Use `when` for conditional rules:
  ```ts
  required(p.field, { when: ({ valueOf }) => valueOf(p.enabled) });
  ```
- Range/format validators should guard against null:
  ```ts
  min(p.count, 1, { when: ({ value }) => value() != null });
  ```
- Cross-field validation: use a top-level rule that reads multiple paths:
  ```ts
  // Not yet in stable API — implement as computed + manual check until supported
  ```

---

## 6. Submitting & Loading State

```ts
export class CustomerFormComponent {
  private customerService = inject(CustomerService);

  model = signal<Customer>(initialCustomer);
  theForm = form(this.model, customerRules);
  isSaving = signal(false);
  saveError = signal<string | null>(null);

  onSubmit(event: SubmitEvent) {
    event.preventDefault();
    if (!this.theForm().valid()) return;

    this.isSaving.set(true);
    this.saveError.set(null);

    this.customerService.save(this.model())
      .pipe(takeUntilDestroyed(this.destroyRef))
      .subscribe({
        next: () => this.isSaving.set(false),
        error: (err) => {
          this.saveError.set(extractErrorMessage(err));
          this.isSaving.set(false);
        },
      });
  }
}
```

---

## 7. Refactor Rules (Converting Old Forms)

Delete:
- `FormGroup`, `FormControl`, `FormBuilder`, `UntypedFormGroup`
- `Validators.*` usages
- `[(ngModel)]`
- `(input)="updateField(...)"` handlers for plain field binding
- `ReactiveFormsModule` from component imports
- Manual `isValid()` / `isInvalid()` helpers

Replace with:
- `signal<TModel>(initial)` + `form(model, rules)`
- `[formField]` bindings
- Schema validators from `@angular/forms/signals`
- `theForm().fieldName().valid()` for validity checks

---

## 8. Reusable Form Controls

For custom inputs that participate in signal forms, implement the signal-forms `ControlValueAccessor` equivalent (the `[formField]` directive protocol). Only fall back to the classic `ControlValueAccessor` for third-party component compatibility.

---

## 9. Code Output Format

When changing form code, always output:
1. File paths at the top of each snippet.
2. Full updated file content (no partial diffs).
3. A short list of removed patterns (e.g. "Removed: FormGroup, Validators.required, [(ngModel)]").
