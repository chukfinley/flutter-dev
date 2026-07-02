# Project structure: feature-first folders

**Rule:** Organize `lib/` **by feature**, not by technical type. Each feature is a
self-contained folder; truly shared code lives in `core/` and `shared/`. Same
layout in every app.

## The problem you asked about

"How do I sort a Flutter repo sensibly so I don't end up with duplicated or
overlooked files?"

The trap is organizing by *type* — one giant `screens/` folder, one giant
`models/`, one giant `widgets/`. In a real app those folders grow to dozens of
unrelated files, you can't tell which belong together, and you end up writing a
second `TodoCard` because you didn't see the first one. Everything related to one
feature is scattered across five top-level folders.

**Feature-first** fixes that: everything for one feature sits in one folder, so
you always know where a thing lives and duplicates become obvious.

## The structure

```
lib/
├── main.dart
├── app.dart                      # root MaterialApp / ProviderScope
├── core/                         # app-wide plumbing, no UI
│   ├── router/                   # go_router table (one place)
│   ├── theme/                    # colors, text styles
│   ├── network/                  # the dio client + interceptors
│   └── db/                       # database setup
├── shared/                       # reused ACROSS features
│   └── widgets/                  # generic buttons, dialogs, etc.
└── features/
    ├── auth/
    │   ├── data/                 # API calls, DB access, DTOs
    │   ├── domain/               # models + business logic (freezed classes)
    │   └── presentation/         # screens + widgets + Riverpod providers
    ├── todos/
    │   ├── data/
    │   ├── domain/
    │   └── presentation/
    └── settings/
        ├── data/
        ├── domain/
        └── presentation/
```

The three layers inside a feature:

- **data** — how you get/save it (dio calls, SQL queries, JSON parsing).
- **domain** — what it *is* (the model classes) and the rules around it.
- **presentation** — what the user sees (screens, widgets) + the Riverpod
  providers that feed them.

Dependencies point **inward**: presentation uses domain, data implements domain.
Domain never imports Flutter.

## Rules of thumb

- **New feature → new folder under `features/`.** Never bolt it onto an existing
  one.
- **Used by exactly one feature → keep it inside that feature.** Only promote a
  widget/util to `shared/` or `core/` once a *second* feature actually needs it.
  (Don't pre-share — that's how you get a `shared/` junk drawer.)
- **Before writing a widget/util, look in that feature's folder and in
  `shared/`.** Feature-first makes the "does this already exist?" check a single
  folder scan, which is how you avoid duplicates.
- **Same names everywhere:** `data / domain / presentation`, `core`, `shared`.
  Identical across all your apps so any project feels familiar instantly.

## Notes / gotchas

- This pairs with the other rules: `core/router/` holds go_router, `core/network/`
  holds the dio client, `domain/` holds `freezed` models, `presentation/` holds
  Riverpod providers.
- For a tiny throwaway app you can flatten it, but defaulting to this structure
  costs nothing and scales without a later reorg.
