# Vue 3 i18n (Internationalization) Agent - Technical Documentation

## Overview

This document describes a comprehensive **i18n (Internationalization) Agent** that enables seamless multi-language support in Vue 3 applications. The agent provides a complete system for managing translations, language switching, and dynamic content rendering in multiple languages.

## Architecture Overview

### Core Components

The i18n Agent consists of four main layers:

```
┌─────────────────────────────────────────────────────┐
│   Vue Components (Template + Script)                │
│   - Uses t() function from useI18n composable       │
│   - Watches language changes for dynamic re-render  │
└────────────────┬────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────┐
│   useI18n Composable                                │
│   - Translation function t(key)                     │
│   - Locale management (setLocale, currentLocale)   │
│   - Fallback mechanism (EN if translation missing)  │
│   - Helper functions (translateProductName, etc)    │
└────────────────┬────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────┐
│   Translation Files (locales/)                      │
│   - en.js (English)                                 │
│   - ja.js (Japanese)                                │
│   - Hierarchical key structure (dot-notation)       │
└────────────────┬────────────────────────────────────┘
                 │
┌────────────────▼────────────────────────────────────┐
│   LocalStorage                                      │
│   - Persists user's language preference             │
│   - Key: 'app-locale'                              │
└─────────────────────────────────────────────────────┘
```

## Detailed Component Specifications

### 1. Translation Files Structure

#### Location
```
src/locales/
├── en.js    (English translations)
└── ja.js    (Japanese translations)
```

#### Format (Nested Object with Dot-Notation Keys)

```javascript
// en.js
export default {
  // Namespace for feature/page
  reports: {
    title: 'Performance Reports',
    description: 'View quarterly performance metrics',
    
    // Nested namespace for sub-feature
    quarterlyPerformance: {
      title: 'Quarterly Performance',
      table: {
        quarter: 'Quarter',
        totalOrders: 'Total Orders',
        totalRevenue: 'Total Revenue',
        avgOrderValue: 'Avg Order Value',
        fulfillmentRate: 'Fulfillment Rate'
      }
    },
    
    // Another sub-section
    monthlyAnalysis: {
      title: 'Month-over-Month Analysis',
      table: {
        month: 'Month',
        orders: 'Orders',
        revenue: 'Revenue'
      }
    },
    
    // Summary section
    summary: {
      totalRevenueYTD: 'Total Revenue (YTD)',
      avgMonthlyRevenue: 'Avg Monthly Revenue',
      totalOrdersYTD: 'Total Orders (YTD)'
    }
  }
}
```

**Key Design Principles:**
- Hierarchical grouping by feature/page
- Descriptive key names (camelCase)
- Logical nesting (3-4 levels deep)
- Plural forms for collections (table, items, etc)

### 2. useI18n Composable API

#### Signature
```javascript
export function useI18n() {
  // Returns object with methods and computed properties
}
```

#### Methods

**`t(key: string, params?: object): string`**
- **Purpose**: Translate a key to the current language
- **Parameters**:
  - `key`: Dot-notation path (e.g., 'reports.title')
  - `params`: Optional object for placeholder replacement (e.g., {count: 5})
- **Returns**: Translated string
- **Behavior**: 
  - Navigates nested object using dot-separated keys
  - Falls back to English if translation missing
  - Returns key itself if translation not found
  - Replaces `{placeholder}` with param values

**Example Usage:**
```javascript
const { t } = useI18n()

// Simple translation
const title = t('reports.title')  // "Performance Reports" or "パフォーマンスレポート"

// With parameters
const message = t('orders.itemsCount', { count: 5 })  // "5 items"

// Nested key access
const quarterLabel = t('reports.quarterlyPerformance.table.quarter')
```

**`setLocale(locale: string): void`**
- **Purpose**: Change the application language
- **Parameters**: 
  - `locale`: Language code (e.g., 'en', 'ja')
- **Side Effects**: 
  - Updates reactive `currentLocale` ref
  - Persists to localStorage under 'app-locale'
  - Triggers reactivity in all components watching currentLocale

**Example Usage:**
```javascript
const { setLocale } = useI18n()

setLocale('ja')  // Switch to Japanese
setLocale('en')  // Switch to English
```

#### Computed Properties

**`currentLocale: Ref<string>`**
- **Type**: Computed ref
- **Returns**: Current language code ('en', 'ja', etc)
- **Reactive**: Yes - changes trigger component re-renders

**`currentCurrency: Computed<string>`**
- **Type**: Computed property
- **Returns**: Currency code based on locale
- **Example**: 'USD' for 'en', 'JPY' for 'ja'
- **Customizable**: Modify currency map in useI18n

**`availableLocales: Computed<string[]>`**
- **Type**: Computed property
- **Returns**: Array of available language codes
- **Example**: ['en', 'ja']

**`localeName: Computed<string>`**
- **Type**: Computed property
- **Returns**: Readable language name in current language
- **Example**: 'English', '日本語'

#### Helper Functions

**`translateProductName(productName: string): string`**
- **Purpose**: Translate product names from data
- **Parameters**: English product name
- **Returns**: Translated product name or original if not found
- **Use Case**: Dynamic product data that needs translation

**`translateCustomerName(customerName: string): string`**
- **Purpose**: Translate customer names from data
- **Similar to**: translateProductName

**`translateWarehouse(warehouseName: string): string`**
- **Purpose**: Translate warehouse/location names
- **Handles**: Both city names (San Francisco → サンフランシスコ) and pattern matching (Warehouse X → 倉庫)
- **Returns**: Translated warehouse name

### 3. Component Integration Pattern

#### Template Usage

```vue
<template>
  <div class="reports">
    <!-- Page header with translations -->
    <div class="page-header">
      <h2>{{ t('reports.title') }}</h2>
      <p>{{ t('reports.description') }}</p>
    </div>

    <!-- Loading state -->
    <div v-if="loading">{{ t('common.loading') }}</div>

    <!-- Table headers -->
    <table>
      <thead>
        <tr>
          <th>{{ t('reports.quarterlyPerformance.table.quarter') }}</th>
          <th>{{ t('reports.quarterlyPerformance.table.totalOrders') }}</th>
        </tr>
      </thead>
    </table>

    <!-- Summary cards -->
    <div class="stat-card">
      <div class="stat-label">{{ t('reports.summary.totalRevenueYTD') }}</div>
      <div class="stat-value">${{ formatNumber(totalRevenue) }}</div>
    </div>
  </div>
</template>
```

#### Script Setup

```javascript
<script>
import { watch } from 'vue'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'ReportsComponent',
  
  setup() {
    // Import i18n composable
    const { t, currentLocale } = useI18n()

    // Watch for language changes and trigger re-render
    watch(currentLocale, () => {
      console.log('Language changed, triggering re-render')
      this.$forceUpdate()  // If using Options API
    })

    return { t }
  },

  // Or using Options API:
  data() {
    return {
      // ... component data
    }
  },

  mounted() {
    const { currentLocale } = useI18n()
    
    // Watch for locale changes
    watch(currentLocale, () => {
      this.$forceUpdate()
    })
  }
}
</script>
```

### 4. Language Switcher Component Pattern

```vue
<template>
  <button @click="toggleMenu" class="language-button">
    <span>{{ localeName }}</span>
    <div v-if="menuOpen" class="language-menu">
      <button 
        v-for="locale in availableLocales" 
        :key="locale"
        @click="setLocale(locale)"
        :class="{ active: currentLocale === locale }"
      >
        {{ getLocaleName(locale) }}
      </button>
    </div>
  </button>
</template>

<script>
import { useI18n } from '../composables/useI18n'
import { ref } from 'vue'

export default {
  setup() {
    const { t, setLocale, currentLocale, availableLocales, localeName } = useI18n()
    const menuOpen = ref(false)

    const getLocaleName = (locale) => {
      const names = {
        en: 'English',
        ja: '日本語'
      }
      return names[locale] || locale
    }

    return {
      setLocale,
      currentLocale,
      availableLocales,
      localeName,
      menuOpen,
      getLocaleName,
      toggleMenu: () => { menuOpen.value = !menuOpen.value }
    }
  }
}
</script>
```

## Implementation Guide

### Step 1: Create Translation Files

Create `src/locales/` directory with translation files:

```
src/locales/
├── en.js
└── ja.js  (or other languages)
```

### Step 2: Define useI18n Composable

Create `src/composables/useI18n.js` with:
- Translation registry
- Locale state management
- Helper functions
- Fallback mechanism

### Step 3: Integrate into Components

In any Vue component:

```javascript
// Import composable
import { useI18n } from '../composables/useI18n'

// Use in setup() or mounted()
const { t, currentLocale } = useI18n()

// Watch for language changes
watch(currentLocale, () => {
  // Re-render component if needed
})
```

### Step 4: Add Language Switcher UI

Create a language selector button component that:
- Shows current language
- Provides dropdown with available languages
- Calls setLocale() when language selected

### Step 5: Test All Languages

- Verify all keys exist in all language files
- Check that special characters render correctly
- Test fallback mechanism (remove a translation and verify English displays)
- Verify localStorage persistence

## Key Features

### 1. **Hierarchical Key Structure**
- Organized by feature/page
- Nested namespaces for related translations
- Easy to maintain and discover keys
- Scalable to many languages

### 2. **Automatic Fallback**
- If translation missing in current language, automatically falls back to English
- Graceful degradation - app always has readable text
- Helps identify missing translations during development

### 3. **Parameter Replacement**
- Support for dynamic values in translations
- Example: `"Welcome {name}"` → `t('welcome', {name: 'John'})`
- Pattern: `{keyName}` in translation, object value in params

### 4. **Persistence**
- Language preference saved to localStorage
- Survives page refreshes
- User's choice remembered across sessions

### 5. **Currency Support**
- Automatically selects currency based on language
- 'USD' for English, 'JPY' for Japanese, etc.
- Customizable mapping

### 6. **Helper Functions**
- `translateProductName()` - Translate dynamic data from API
- `translateCustomerName()` - Translate customer names
- `translateWarehouse()` - Smart warehouse/location translation
- Extensible pattern for custom translators

## Usage Patterns

### Pattern 1: Simple Page Translation

```javascript
const { t } = useI18n()

// In template
{{ t('reports.title') }}
{{ t('reports.description') }}
```

### Pattern 2: Table Headers

```javascript
const tableHeaders = computed(() => [
  { key: 'quarter', label: t('reports.quarterlyPerformance.table.quarter') },
  { key: 'orders', label: t('reports.quarterlyPerformance.table.totalOrders') },
  { key: 'revenue', label: t('reports.quarterlyPerformance.table.totalRevenue') }
])
```

### Pattern 3: Dynamic Content Translation

```javascript
const { translateProductName, t } = useI18n()

// Translate product from API data
const displayName = translateProductName(apiProduct.name)
```

### Pattern 4: Error Messages

```javascript
const { t } = useI18n()

try {
  // operation
} catch (err) {
  alert(t('common.error'))
}
```

### Pattern 5: Conditional UI Based on Language

```javascript
const { currentLocale } = useI18n()

// Show/hide content based on language
const showJapaneseOnlyFeature = computed(() => {
  return currentLocale.value === 'ja'
})
```

## Best Practices

### ✅ Do's

1. **Organize translations hierarchically**
   - Group by feature/page
   - Use descriptive key names

2. **Use parameter replacement for dynamic content**
   - `"Total: {count} items"` instead of string concatenation

3. **Watch currentLocale for dynamic content**
   - Ensure component re-renders when language changes

4. **Test all language combinations**
   - Verify layout works with translations
   - Account for text length differences

5. **Document complex translations**
   - Add comments for context-dependent strings

6. **Keep translations up-to-date**
   - Don't mark incomplete translations with TODO in code
   - Provide fallback immediately

### ❌ Don'ts

1. **Don't mix locales in components**
   - Don't hardcode strings alongside t() calls
   - Use t() for ALL user-visible text

2. **Don't rely on hardcoded language codes**
   - Use availableLocales from useI18n
   - Allows easy addition of new languages

3. **Don't forget to handle missing translations**
   - Always provide English fallback
   - Test with missing keys

4. **Don't translate fixed data from API**
   - Some data (like product IDs) shouldn't be translated
   - Only translate display labels, UI text

5. **Don't assume English word order**
   - Some languages have different grammar rules
   - Avoid string concatenation - use full translations

## Performance Considerations

### 1. **Translation Lookup Time**
- O(n) where n = depth of nested key
- Typically 3-4 levels deep = negligible performance impact
- Cache if needed using computed properties

### 2. **Reactivity**
- Only currentLocale changes trigger reactivity
- Translation strings themselves are static
- Efficient for large numbers of translations

### 3. **Bundle Size**
- Each language file adds to bundle
- Consider lazy-loading additional languages
- Typical translation file: 10-50 KB

### 4. **Memory Usage**
- All language files loaded in memory
- Not suitable for >10 large language files
- Consider splitting by module if needed

## Extending the System

### Adding a New Language

1. **Create new translation file:**
```javascript
// src/locales/fr.js
export default {
  nav: {
    overview: 'Aperçu',
    // ... all keys
  }
}
```

2. **Update useI18n.js:**
```javascript
import fr from '../locales/fr'

const translations = {
  en,
  ja,
  fr  // Add here
}
```

3. **Update language names:**
```javascript
const names = {
  en: 'English',
  ja: '日本語',
  fr: 'Français'  // Add here
}
```

### Adding Custom Helper Functions

```javascript
// In useI18n.js
const translateCustomField = (value) => {
  if (currentLocale.value === 'ja') {
    return customTranslations.ja[value] || value
  }
  return value
}

// Export alongside other helpers
return {
  t,
  setLocale,
  // ... other exports
  translateCustomField
}
```

### Adding Currency Symbol Translation

```javascript
const currentCurrencySymbol = computed(() => {
  const symbols = {
    USD: '$',
    JPY: '¥',
    EUR: '€'
  }
  return symbols[currentCurrency.value]
})
```

## Testing Strategies

### Unit Tests

```javascript
import { useI18n } from '@/composables/useI18n'

describe('useI18n', () => {
  it('should translate keys correctly', () => {
    const { t, setLocale } = useI18n()
    
    setLocale('en')
    expect(t('reports.title')).toBe('Performance Reports')
    
    setLocale('ja')
    expect(t('reports.title')).toBe('パフォーマンスレポート')
  })

  it('should fallback to English', () => {
    const { t, setLocale } = useI18n()
    
    setLocale('ja')
    // If translation missing, should return English
    expect(t('missing.key')).toBeDefined()
  })

  it('should replace parameters', () => {
    const { t } = useI18n()
    expect(t('orders.itemsCount', { count: 5 })).toBe('5 items')
  })
})
```

### Integration Tests

```javascript
it('should persist language preference', async () => {
  const { setLocale, currentLocale } = useI18n()
  
  setLocale('ja')
  expect(localStorage.getItem('app-locale')).toBe('ja')
  
  // Refresh and verify
  const newInstance = useI18n()
  expect(newInstance.currentLocale.value).toBe('ja')
})
```

### E2E Tests

```javascript
// Test language switching in UI
cy.visit('/reports')
cy.get('[data-test=language-button]').click()
cy.get('[data-test=language-ja]').click()

// Verify page headers changed
cy.get('h2').should('contain', 'パフォーマンスレポート')
cy.get('th').first().should('contain', '四半期')
```

## Migration Guide from Hardcoded Strings

### Before (Hardcoded):
```vue
<template>
  <div>
    <h2>Performance Reports</h2>
    <table>
      <th>Quarter</th>
      <th>Total Orders</th>
    </table>
  </div>
</template>
```

### After (Internationalized):
```vue
<template>
  <div>
    <h2>{{ t('reports.title') }}</h2>
    <table>
      <th>{{ t('reports.quarterlyPerformance.table.quarter') }}</th>
      <th>{{ t('reports.quarterlyPerformance.table.totalOrders') }}</th>
    </table>
  </div>
</template>

<script>
import { useI18n } from '@/composables/useI18n'

export default {
  setup() {
    const { t } = useI18n()
    return { t }
  }
}
</script>
```

## Troubleshooting

### Issue: Translations not updating when language changes
**Solution:** Watch `currentLocale` and trigger re-render:
```javascript
const { currentLocale } = useI18n()
watch(currentLocale, () => this.$forceUpdate())
```

### Issue: Missing translations showing as keys
**Solution:** Add missing key to all language files or add to English as fallback

### Issue: Text not fitting in UI after translation
**Solution:** Use flexible layouts (flex, grid) that adapt to text length

### Issue: Special characters not displaying correctly
**Solution:** Ensure .js files are saved as UTF-8, not ASCII

### Issue: LocalStorage not persisting preference
**Solution:** Check browser privacy settings, not in private/incognito mode

## Real-World Example: Reports Page

### Translation File Structure
```javascript
// locales/en.js
reports: {
  title: 'Performance Reports',
  description: 'View quarterly performance metrics and monthly trends',
  quarterlyPerformance: {
    title: 'Quarterly Performance',
    table: {
      quarter: 'Quarter',
      totalOrders: 'Total Orders',
      totalRevenue: 'Total Revenue',
      avgOrderValue: 'Avg Order Value',
      fulfillmentRate: 'Fulfillment Rate'
    }
  },
  monthlyRevenueTrend: {
    title: 'Monthly Revenue Trend'
  },
  monthlyAnalysis: {
    title: 'Month-over-Month Analysis',
    table: {
      month: 'Month',
      orders: 'Orders',
      revenue: 'Revenue',
      change: 'Change',
      growthRate: 'Growth Rate'
    }
  },
  summary: {
    totalRevenueYTD: 'Total Revenue (YTD)',
    avgMonthlyRevenue: 'Avg Monthly Revenue',
    totalOrdersYTD: 'Total Orders (YTD)',
    bestPerformingQuarter: 'Best Performing Quarter'
  }
}

// locales/ja.js (Japanese)
reports: {
  title: 'パフォーマンスレポート',
  description: '四半期別パフォーマンス指標と月別トレンドを表示',
  quarterlyPerformance: {
    title: '四半期別パフォーマンス',
    table: {
      quarter: '四半期',
      totalOrders: '総注文数',
      totalRevenue: '総収益',
      avgOrderValue: '平均注文額',
      fulfillmentRate: '履行率'
    }
  },
  // ... more translations
}
```

### Component Implementation
```vue
<template>
  <div class="reports">
    <div class="page-header">
      <h2>{{ t('reports.title') }}</h2>
      <p>{{ t('reports.description') }}</p>
    </div>

    <div v-if="loading">{{ t('common.loading') }}</div>
    
    <div v-else>
      <div class="card">
        <h3>{{ t('reports.quarterlyPerformance.title') }}</h3>
        <table>
          <thead>
            <tr>
              <th>{{ t('reports.quarterlyPerformance.table.quarter') }}</th>
              <th>{{ t('reports.quarterlyPerformance.table.totalOrders') }}</th>
              <!-- ... more headers -->
            </tr>
          </thead>
        </table>
      </div>

      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.summary.totalRevenueYTD') }}</div>
          <div class="stat-value">${{ formatNumber(totalRevenue) }}</div>
        </div>
        <!-- ... more stat cards -->
      </div>
    </div>
  </div>
</template>

<script>
import { watch } from 'vue'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Reports',
  
  setup() {
    const { t, currentLocale } = useI18n()

    // Watch for language changes
    watch(currentLocale, () => {
      console.log('Language updated')
    })

    return { t }
  },

  data() {
    return {
      loading: true,
      quarterlyData: [],
      totalRevenue: 0
    }
  }
}
</script>
```

## Summary

This i18n Agent provides:
- ✅ **Modular**: Composable-based architecture, easy to import and use
- ✅ **Scalable**: Supports unlimited languages and translation keys
- ✅ **Performant**: Minimal overhead, efficient lookup mechanism
- ✅ **Flexible**: Fallback system, parameter replacement, helper functions
- ✅ **Persistent**: Saves user language preference
- ✅ **Maintainable**: Hierarchical key structure, organized translation files
- ✅ **Reactive**: Automatic re-renders when language changes
- ✅ **Extensible**: Easy to add new languages, custom translators, features

Perfect for any Vue 3 application needing multi-language support!
