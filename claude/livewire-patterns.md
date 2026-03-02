# Livewire Patterns & Troubleshooting

> Reference file for CLAUDE.md - Component structure, modal patterns, Alpine.js, hydration issues

## Component Structure Rules

**CRITICAL**: Follow these patterns to avoid component lifecycle & JavaScript errors

### Structure Requirements

**NEVER nest components too deeply** - Livewire breaks with excessive wrapper divs:
```blade
<!-- WRONG: 4+ levels of nesting breaks Livewire -->
<div>
    <style>...</style>
    <div class="container">
        <div class="flex justify-center">
            <div class="w-full max-w-6xl">...</div>
        </div>
    </div>
</div>

<!-- CORRECT: 2 levels maximum -->
<div>
    <div class="max-w-6xl mx-auto py-6 px-4 sm:px-6 lg:px-8">
        <!-- Content -->
    </div>
    <style>...</style>
</div>
```

### JavaScript Event Listeners with wire:navigate

**NEVER use `DOMContentLoaded`** - Only fires once, causing "second search fails" bugs:
```javascript
// WRONG:
document.addEventListener('DOMContentLoaded', function () { ... });

// CORRECT:
document.addEventListener('livewire:navigated', function () { ... });
```

### Component Listeners

**NEVER use `'$refresh'` magic string** - Breaks component state:
```php
// WRONG:
protected $listeners = ['customerAdded' => '$refresh'];

// CORRECT:
protected $listeners = ['customerAdded' => 'refreshComponent'];
public function refreshComponent() { $this->resetPage(); }
```

### Root Element Rules

**ALWAYS have exactly ONE root element** - Multiple roots break Livewire:
```blade
<!-- WRONG: @livewire creates second root element -->
<div>
    <style>...</style>
    @livewire('modal-component')  <!-- Second root! -->
    <div class="content">...</div>
</div>

<!-- CORRECT: @livewire inside root element -->
<div>
    <div class="content">
        @livewire('modal-component')
    </div>
    <style>...</style>
</div>
```

### Common Error Patterns

- **"window.livewireScriptConfig.uri" undefined**: Excessive nesting, DOMContentLoaded, $refresh, or multiple root elements
- **"First search works, second fails"**: DOMContentLoaded not re-running → use `livewire:navigated`
- **Modals stop working after first interaction**: Multiple root elements or $refresh

## Modal Best Practices

### CORRECT Pattern (Like ProductEdit/EditTagForm)

**1. Separate Modal Component File**:
```php
class EditProductForm extends Component {
    public $productId = null;
    public $product = null;
    public $name = '';
    public $price = 0;
    // NO $queryString here!

    public function mount($productId = null) {
        if ($productId) { $this->loadProduct($productId); }
    }
    public function save() {
        // Save logic
        $this->dispatch('product-saved');
    }
}
```

**2. Parent Component (List/Table)**:
```php
class ProductTable extends Component {
    public $editModalStates = [];
    protected $listeners = ['product-saved' => 'refreshList'];

    public function editProduct($id) { $this->editModalStates[$id] = true; }
    public function refreshList() {
        $this->editModalStates = [];
        $this->resetPage();
        // DO NOT use $this->dispatch('$refresh')!
    }
}
```

**3. Parent Blade Template**:
```blade
@foreach($products as $product)
    <flux:modal wire:model.self="editModalStates.{{ $product->id }}">
        @livewire('edit-product-form', ['productId' => $product->id], key('edit-product-' . $product->id))
    </flux:modal>
@endforeach
```

### Key Rules

1. Modal forms MUST be separate Livewire components
2. Nested modal components should NEVER have $queryString
3. Load data in `mount($id)`, not in parent component
4. Use `wire:model.self` modifier to prevent event bubbling
5. Use `wire:ignore` for static elements Livewire shouldn't morph
6. Reset state arrays instead of using $refresh
7. Always use unique `key()` for each modal instance
8. Use `$this->dispatch()` for component communication
9. Track open/closed state with `$editModalStates[$id]` pattern
10. Use `setTimeout(..., 100)` when showing modals via JavaScript

### Troubleshooting

- **"Snapshot missing" error**: Used `$wire.$refresh()` → reset state arrays
- **Query params in URL**: Nested component has `$queryString` → remove it
- **Form fields don't populate**: Data loaded in parent instead of nested `mount()`
- **Can't reopen modal**: Modal states not reset after save
- **DOM errors with buttons**: Use `wire:ignore` + Alpine for static interactive elements

## Livewire v3 + Alpine.js Setup

**CRITICAL**: Livewire v3 includes Alpine.js automatically. External Alpine scripts cause conflicts.

### Correct Setup

```blade
<head>
    @vite(['resources/css/app.css', 'resources/js/app.js'])
    @fluxAppearance
    @livewireStyles
</head>
<body>
    {{ $slot }}
    @livewireScripts  <!-- Includes Alpine.js automatically -->
    @fluxScripts
</body>
```

### NEVER DO THIS
- Manually load Alpine.js via CDN or npm
- Add `window.livewireScriptConfig` manually
- Import Alpine in resources/js/app.js

## Livewire Hydration Issues

**Problem**: Eloquent model casts are lost when models are stored in Livewire public properties between requests.

**Example**: `$subscription->custom_fields` (array) becomes JSON string after hydration.

**Solution**: Create helper methods:
```php
public function getCustomFieldsArray()
{
    $customFields = $this->subscription->custom_fields;
    if (is_string($customFields)) {
        $customFields = json_decode($customFields, true) ?? [];
    }
    return is_array($customFields) ? $customFields : [];
}
```

## Troubleshooting Checklist

When modals/Alpine components stop working:

1. **Check browser cache**: Hard refresh (Cmd+Shift+R / Ctrl+Shift+F5)
2. **Verify file permissions**: Build files must be `www-data:www-data`, not `root:root`
3. **Check for external Alpine scripts**: Remove ALL Alpine CDN/npm imports
4. **Verify script loading order**: `@livewireScripts` before `@fluxScripts`
5. **Clear Laravel caches**: `php artisan config:clear && php artisan view:clear && npm run build`
6. **Check browser console**: Look for "Alpine is not defined" or "livewireScriptConfig" errors

## Incident History (Never Forget)

**October 11, 2025**: Tag Edit Modal + Custom Fields Errors
- Custom fields hydration bug (array → string via Livewire hydration)
- Browser cache serving old JavaScript builds
- File permission issues (root-owned build files)
- **Key Lesson**: Trust framework defaults. Livewire v3 bundles Alpine automatically. External Alpine scripts create conflicts, not solutions.
