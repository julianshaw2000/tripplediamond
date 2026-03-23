# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Triple Diamond Multi Service** — an e-commerce platform for organic handmade soaps and skincare products. UK and Nigeria based brand selling handmade soaps, organic shea butter, face oils, body butters, luxury skincare sets, and organic honey.

## Planned Architecture

Monorepo with three packages:

- **packages/api** — ASP.NET Core (.NET 9+) backend using Minimal APIs, MediatR (CQRS), EF Core, FluentValidation, Result pattern
- **packages/web** — Angular 21+ frontend with standalone components, signal-based state, signal forms
- **packages/shared** — Shared TypeScript types/utilities consumed by the web package
- **packages/worker** — .NET background worker service

## Build & Test Commands

### .NET API
```bash
dotnet build packages/api/src/Tungsten.Api/
dotnet test packages/api/tests/Tungsten.Api.Tests/
dotnet format packages/api/ --verify-no-changes   # CI lint check
```

### Angular Web
```bash
cd packages/web && npm install
cd packages/web && npx ng serve          # dev server
cd packages/web && npx ng test           # unit tests
cd packages/web && npx ng build          # production build
```

### CI Gate
```bash
dotnet build && dotnet test && dotnet format --verify-no-changes
```

## .NET Coding Rules

- **Vertical Slice Architecture**: organize by feature (`Features/<Feature>/`), not by layer. Cross-slice sharing goes in `Common/` or `Infrastructure/`.
- **CQRS via MediatR**: every use case is `IRequest<TResponse>` + `IRequestHandler`. Commands return `Result<T>`, queries return DTOs (never entities).
- **Result pattern, not exceptions**: use `Result<T>` for expected failures. Reserve exceptions for truly exceptional conditions. Map Result → HTTP only in the endpoint layer.
- **Minimal APIs preferred**: group endpoints by feature via `IEndpointRouteBuilder` extensions. Use `TypedResults` for compile-time checked responses.
- **FluentValidation** with MediatR `ValidationBehaviour` pipeline. Validators live next to their request type.
- **Primary constructors** (C# 12+) for services. `record` for DTOs/commands/queries.
- **EF Core**: `AsNoTracking()` on reads, project to DTOs directly, never return entities from APIs. Scoped `DbContext`, never singleton.
- **Async everywhere**: no `.Result`, `.Wait()`, or `GetAwaiter().GetResult()`. Always forward `CancellationToken`.
- Secrets from environment variables or secrets manager, never from checked-in `appsettings.json`.

## Angular Coding Rules

- **Standalone-first**: all components `standalone: true`, no NgModules. Lazy-load all feature routes.
- **Signal-based state**: `signal()` for state, `computed()` for derived values. No `Subject` for state, no `async` pipe.
- **Signal APIs only**: `input()` / `output()` / `model()` (not `@Input`/`@Output`), `viewChild()` / `contentChild()` (not decorators), `inject()` (not constructor DI).
- **Smart/Dumb component pattern**: smart components own state and inject facades; dumb components use `input()`/`output()` only with `OnPush` change detection.
- **Facade pattern**: one facade per feature wraps store + API. Components never touch stores or API services directly.
- **Signal Forms** (`@angular/forms/signals`): `signal<TModel>` + `form(model, rules)` + `[formField]` binding. No `FormGroup`, `FormControl`, `[(ngModel)]`, or `ReactiveFormsModule`.
- **Template control flow**: `@if`/`@for`/`@switch` only, never `*ngIf`/`*ngFor`.
- **HTTP in data services only**: `httpResource()` for reactive GETs, `HttpClient` for mutations. Adapter functions transform DTOs → domain models at the boundary.
- **Feature structure**: `features/<feature>/data/` (API services, models, adapters), `features/<feature>/ui/` (presentational components), facade and store at feature root.
- **Dependency rules**: `shared/` → Angular only; `core/` → Angular + shared; `features/` → core + shared. Never cross-import between features.

## File Naming

- .NET: PascalCase filenames matching type name, file-scoped namespaces
- Angular: kebab-case filenames (`order-list.component.ts`), PascalCase classes with type suffix
