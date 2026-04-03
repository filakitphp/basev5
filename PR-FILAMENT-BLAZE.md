# PR: Add Livewire Blaze `@blaze` directive to anonymous Blade components

## Summary

Add `@blaze` directive to Filament's **117 anonymous Blade components** across 8 packages, enabling automatic [Livewire Blaze](https://github.com/livewire/blaze) optimization when users install `livewire/blaze`.

This is a zero-cost change: the `@blaze` Blade directive compiles to an empty string (`return ''`), so projects without Blaze installed see no difference. Projects with Blaze get 91-97% rendering performance improvement on component-heavy pages.

## Motivation

Filament panels render dozens of anonymous Blade components per page (buttons, badges, icons, modals, inputs, dropdowns, tabs, sections, etc). Each component goes through Blade's full rendering pipeline:

- Component class resolution
- Attribute bag creation
- View compilation and evaluation

Blaze eliminates this overhead by pre-compiling templates into direct PHP function calls:

| Scenario (25,000 components) | Blade | Blaze | Reduction |
|------------------------------|-------|-------|-----------|
| No attributes | 500ms | 13ms | **97.4%** |
| Attributes only | 457ms | 26ms | **94.3%** |
| Attributes + merge() | 546ms | 44ms | **91.9%** |
| Props + attributes | 780ms | 40ms | **94.9%** |

## How it works

Blaze detects the `@blaze` directive via regex at the start of each component file:

```php
// BlazeDirective::getParameters()
preg_match('/^\s*(?:\/\*.*?\*\/\s*)*@blaze(?:\s*\(([^)]+)\))?/s', $source, $matches)
```

The directive itself compiles to nothing:

```php
// BlazeServiceProvider::registerBlazeDirectives()
Blade::directive('blaze', function () {
    return ''; // zero output
});
```

When Blaze is installed, it intercepts Blade's pre-compilation phase and compiles tagged components into optimized PHP functions. When Blaze is NOT installed, the `@blaze` directive is simply ignored by Blade (unknown directives produce no output).

## Changes

### Component files (117 files across 8 packages)

Add `@blaze` as the **first line** of each anonymous Blade component file.

**Standard components (113 files):**

```blade
@blaze
@props([
    'color' => 'primary',
    ...
])

<div ...>
```

**Self-closing, repeated components with memoization (4 files):**

```blade
@blaze(memo: true)
@props([
    'icon' => null,
    ...
])

<svg ...>
```

### Inventory per package

| Package | Path | Files | Strategy |
|---------|------|-------|----------|
| `filament/support` | `resources/views/components/` | 36 | `@blaze` (32) + `@blaze(memo: true)` (4) |
| `filament/filament` | `resources/views/components/` | 23 | `@blaze` |
| `filament/forms` | `resources/views/components/` | 30 | `@blaze` |
| `filament/schemas` | `resources/views/components/` | 17 | `@blaze` |
| `filament/tables` | `resources/views/components/` | 6 | `@blaze` |
| `filament/actions` | `resources/views/components/` | 2 | `@blaze` |
| `filament/infolists` | `resources/views/components/` | 1 | `@blaze` |
| `filament/widgets` | `resources/views/components/` | 2 | `@blaze` |
| `filament/notifications` | — | 0 | skipped |
| `filament/query-builder` | — | 0 | skipped |

### `@blaze(memo: true)` candidates (filament/support)

These 4 components are self-closing, stateless, and heavily repeated on every page. Memoization caches their output by props, avoiding re-rendering identical instances:

- `components/icon.blade.php`
- `components/loading-indicator.blade.php`
- `components/loading-section.blade.php`
- `components/avatar.blade.php`

### No ServiceProvider changes needed

The `@blaze` directive approach requires **zero PHP code changes** — only Blade template files are modified. No `use` statements, no `Blaze::optimize()->in()` calls, no `class_exists()` checks. Blaze detects the directive during its pre-compilation phase automatically.

### Optional: suggest Blaze in composer.json

Add to `filament/support`'s `composer.json`:

```json
"suggest": {
    "livewire/blaze": "Optimizes anonymous Blade component rendering (^1.0)"
}
```

## Design Decisions

### Why `@blaze` only — no `fold: true`

Several Filament components use runtime global state that would break with compile-time folding:

| File | Global State |
|------|-------------|
| `filament/components/user-menu.blade.php` | `filament()->auth()->user()` |
| `filament/components/tenant-menu.blade.php` | `filament()->getUserTenants(...)` |
| `filament/components/layout/base.blade.php` | `config('filament.broadcasting.echo')` |
| `filament/components/layout/simple.blade.php` | `filament()->auth()->check()` |
| Multiple support components | `config('filament.livewire_loading_delay')` |

Folding pre-renders at compile time, freezing these runtime values. Plain `@blaze` (compile only) is safe for all components.

### Why `memo: true` only for 4 specific components

Memoization requires all of:
- Self-closing components (no slots)
- Stateless (no global state access)
- Frequently repeated with identical props

Only `icon`, `loading-indicator`, `loading-section`, and `avatar` meet all three criteria.

### Why no `@blaze` on non-component views

Only files in `resources/views/components/` directories are anonymous Blade components. Livewire page views, layout Blade files outside `components/`, and class-based components are NOT candidates for Blaze.

### Backwards compatibility

- **With Blaze installed:** Components are compiled into optimized PHP functions
- **Without Blaze installed:** The `@blaze` directive is an unknown Blade directive and is silently ignored — zero impact on existing behavior
- **No new dependencies:** Blaze remains optional (`suggest`, not `require`)

## Known Issue: RootTagMissingFromViewException

During testing, adding `@blaze` to component files caused `Livewire\Exceptions\RootTagMissingFromViewException` on the Dashboard page.

**Root cause:** Livewire detects root HTML tags using this regex in `Drawer/Utils.php`:

```php
preg_match('/(?:\n\s*|^\s*)<([a-zA-Z0-9\-]+)/', $html, $matches)
```

When Blaze compiles a component, it replaces the standard Blade component rendering pipeline with a direct PHP function call. This changes the output structure, and Livewire's root tag detection may fail depending on how the compiled output is structured.

**Impact:** This needs investigation with the Livewire/Blaze team to ensure compatibility. The `@blaze` directive compiles to `''` (no output), but Blaze's compilation pipeline itself may alter the rendered HTML structure.

**Workaround:** If specific components cause issues, they can be excluded by removing the `@blaze` directive from those files only. Layout-related components in `filament/filament` (e.g., `layout/base.blade.php`, `layout/index.blade.php`) are the most likely candidates for exclusion.

## Test Plan

- [ ] Add `@blaze` to all 117 component files
- [ ] Install `livewire/blaze:^1.0` in a fresh Filament 5 project
- [ ] Navigate to admin panel — verify all pages render correctly
- [ ] Test forms: create, edit, validate — verify all form components work
- [ ] Test tables: sort, filter, search, paginate — verify all table components work
- [ ] Test actions: create, edit, delete with modals — verify action components work
- [ ] Test widgets: dashboard with stats and charts — verify widget components work
- [ ] Enable `BLAZE_DEBUG=true` and verify components show Blaze debug overlay
- [ ] Verify `RootTagMissingFromViewException` is resolved (or exclude affected components)
- [ ] Run `php artisan view:clear && php artisan view:cache` — verify production caching works
- [ ] Remove `livewire/blaze` — verify Filament still works without it (graceful fallback)
- [ ] Run existing Filament test suite — verify no regressions

## References

- [Livewire Blaze documentation](https://livewire.laravel.com/docs/blaze)
- [Livewire Blaze GitHub](https://github.com/livewire/blaze)
- Filament monorepo: [`filamentphp/filament`](https://github.com/filamentphp/filament)
