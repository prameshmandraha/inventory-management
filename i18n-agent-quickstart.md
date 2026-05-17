# Vue 3 i18n Agent - Quick Start Guide

## Installation & Setup (5 minutes)

### Step 1: Create Translation Files

```
src/locales/
├── en.js    # English translations
└── ja.js    # Japanese translations (or other languages)
```

**en.js:**
```javascript
export default {
  nav: {
    overview: 'Overview',
    reports: 'Reports'
  },
  reports: {
    title: 'Performance Reports',
    description: 'View quarterly performance metrics',
    quarterlyPerformance: {
      title: 'Quarterly Performance',
      table: {
        quarter: 'Quarter',
        totalOrders: 'Total Orders',
        totalRevenue: 'Total Revenue'
      }
    },
    summary: {
      totalRevenueYTD: 'Total Revenue (YTD)',
      avgMonthlyRevenue: 'Avg Monthly Revenue'
    }
  },
  common: {
    loading: 'Loading...',
    error: 'Error'
  }
}
```

**ja.js:**
```javascript
export default {
  nav: {
    overview: '概要',
    reports: 'レポート'
  },
  reports: {
    title: 'パフォーマンスレポート',
    description: '四半期別パフォーマンス指標を表示',
    quarterlyPerformance: {
      title: '四半期別パフォーマンス',
      table: {
        quarter: '四半期',
        totalOrders: '総注文数',
        totalRevenue: '総収益'
      }
    },
    summary: {
      totalRevenueYTD: '総収益（年初来）',
      avgMonthlyRevenue: '平均月次収益'
    }
  },
  common: {
    loading: '読み込み中...',
    error: 'エラー'
  }
}
```

### Step 2: Create useI18n Composable

**src/composables/useI18n.js:**
```javascript
import { ref, computed } from 'vue'
import en from '../locales/en'
import ja from '../locales/ja'

const translations = { en, ja }
const savedLocale = localStorage.getItem('app-locale') || 'en'
const currentLocale = ref(savedLocale)

const currentCurrency = computed(() => {
  return currentLocale.value === 'ja' ? 'JPY' : 'USD'
})

export function useI18n() {
  const t = (key, params = {}) => {
    const keys = key.split('.')
    let value = translations[currentLocale.value]

    for (const k of keys) {
      if (value && typeof value === 'object') {
        value = value[k]
      } else {
        // Fallback to English
        if (currentLocale.value !== 'en') {
          let fallback = translations.en
          for (const fk of keys) {
            if (fallback && typeof fallback === 'object') {
              fallback = fallback[fk]
            } else {
              break
            }
          }
          if (fallback && typeof fallback === 'string') {
            return replacePlaceholders(fallback, params)
          }
        }
        return key
      }
    }

    if (typeof value === 'string') {
      return replacePlaceholders(value, params)
    }

    return key
  }

  const replacePlaceholders = (text, params) => {
    return text.replace(/\{(\w+)\}/g, (match, key) => {
      return params[key] !== undefined ? params[key] : match
    })
  }

  const setLocale = (locale) => {
    if (translations[locale]) {
      currentLocale.value = locale
      localStorage.setItem('app-locale', locale)
    }
  }

  return {
    t,
    setLocale,
    currentLocale: computed(() => currentLocale.value),
    currentCurrency,
    availableLocales: computed(() => Object.keys(translations))
  }
}
```

### Step 3: Create Language Switcher Component

**src/components/LanguageSwitcher.vue:**
```vue
<template>
  <div class="language-switcher">
    <button @click="showMenu = !showMenu" class="lang-button">
      {{ currentLocale.toUpperCase() }}
      <svg class="chevron" :class="{ open: showMenu }" viewBox="0 0 20 20" fill="currentColor">
        <path fill-rule="evenodd" d="M5.293 7.293a1 1 0 011.414 0L10 10.586l3.293-3.293a1 1 0 111.414 1.414l-4 4a1 1 0 01-1.414 0l-4-4a1 1 0 010-1.414z" clip-rule="evenodd" />
      </svg>
    </button>

    <div v-if="showMenu" class="lang-menu">
      <button 
        v-for="locale in availableLocales" 
        :key="locale"
        @click="selectLanguage(locale)"
        :class="{ active: currentLocale === locale }"
      >
        {{ getLanguageName(locale) }}
      </button>
    </div>
  </div>
</template>

<script>
import { ref } from 'vue'
import { useI18n } from '../composables/useI18n'

export default {
  setup() {
    const { setLocale, currentLocale, availableLocales } = useI18n()
    const showMenu = ref(false)

    const getLanguageName = (locale) => {
      const names = {
        en: 'English',
        ja: '日本語'
      }
      return names[locale] || locale
    }

    const selectLanguage = (locale) => {
      setLocale(locale)
      showMenu.value = false
    }

    return {
      currentLocale,
      availableLocales,
      showMenu,
      getLanguageName,
      selectLanguage
    }
  }
}
</script>

<style scoped>
.language-switcher {
  position: relative;
}

.lang-button {
  padding: 0.5rem 1rem;
  background: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  cursor: pointer;
  display: flex;
  align-items: center;
  gap: 0.5rem;
}

.chevron {
  width: 20px;
  height: 20px;
  transition: transform 0.2s;
}

.chevron.open {
  transform: rotate(180deg);
}

.lang-menu {
  position: absolute;
  top: 100%;
  left: 0;
  background: white;
  border: 1px solid #ccc;
  border-radius: 4px;
  min-width: 150px;
  z-index: 10;
  margin-top: 0.5rem;
}

.lang-menu button {
  display: block;
  width: 100%;
  padding: 0.75rem 1rem;
  border: none;
  background: none;
  text-align: left;
  cursor: pointer;
  font-size: 0.875rem;
}

.lang-menu button:hover {
  background: #f5f5f5;
}

.lang-menu button.active {
  background: #e0e0e0;
  font-weight: 600;
}
</style>
```

## Basic Usage

### In Vue Components

**Simple Translation:**
```vue
<template>
  <div>
    <h1>{{ t('reports.title') }}</h1>
    <p>{{ t('reports.description') }}</p>
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

**Table Headers:**
```vue
<template>
  <table>
    <thead>
      <tr>
        <th>{{ t('reports.quarterlyPerformance.table.quarter') }}</th>
        <th>{{ t('reports.quarterlyPerformance.table.totalOrders') }}</th>
        <th>{{ t('reports.quarterlyPerformance.table.totalRevenue') }}</th>
      </tr>
    </thead>
  </table>
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

**With Loading State:**
```vue
<template>
  <div>
    <div v-if="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error">{{ t('common.error') }}</div>
    <div v-else>
      <h2>{{ t('reports.title') }}</h2>
      <!-- Content -->
    </div>
  </div>
</template>

<script>
import { useI18n } from '@/composables/useI18n'

export default {
  data() {
    return {
      loading: true,
      error: null
    }
  },

  setup() {
    const { t } = useI18n()
    return { t }
  }
}
</script>
```

**Watch Language Changes:**
```javascript
import { watch } from 'vue'
import { useI18n } from '@/composables/useI18n'

export default {
  setup() {
    const { t, currentLocale } = useI18n()

    // Re-render when language changes
    watch(currentLocale, () => {
      console.log('Language changed!')
      // Component automatically re-renders
    })

    return { t }
  }
}
</script>
```

### With Parameters (Dynamic Content)

**Translation File:**
```javascript
// en.js
orders: {
  itemsCount: '{count} items',
  welcome: 'Welcome {name}!'
}

// ja.js
orders: {
  itemsCount: '{count}個のアイテム',
  welcome: '{name}さんへようこそ！'
}
```

**Component Usage:**
```javascript
const { t } = useI18n()

const message1 = t('orders.itemsCount', { count: 5 })
// English: "5 items"
// Japanese: "5個のアイテム"

const message2 = t('orders.welcome', { name: 'John' })
// English: "Welcome John!"
// Japanese: "Johnさんへようこそ！"
```

## Common Patterns

### Pattern 1: Translate Table Headers

```javascript
const columns = computed(() => [
  { key: 'quarter', label: t('reports.quarterlyPerformance.table.quarter') },
  { key: 'orders', label: t('reports.quarterlyPerformance.table.totalOrders') },
  { key: 'revenue', label: t('reports.quarterlyPerformance.table.totalRevenue') }
])
```

### Pattern 2: Translate List Items

```javascript
const statCards = computed(() => [
  {
    label: t('reports.summary.totalRevenueYTD'),
    value: totalRevenue
  },
  {
    label: t('reports.summary.avgMonthlyRevenue'),
    value: avgMonthly
  }
])
```

### Pattern 3: Format with Currency

```javascript
import { useI18n } from '@/composables/useI18n'

export default {
  setup() {
    const { t, currentCurrency } = useI18n()

    const formatPrice = (price) => {
      const symbol = currentCurrency.value === 'JPY' ? '¥' : '$'
      return `${symbol}${price.toLocaleString()}`
    }

    return { t, formatPrice }
  }
}
```

## Translation Structure Best Practices

### Good Structure:
```javascript
reports: {
  title: 'Performance Reports',
  description: 'View quarterly performance metrics',
  
  quarterlyPerformance: {
    title: 'Quarterly Performance',
    table: {
      quarter: 'Quarter',
      totalOrders: 'Total Orders'
    }
  },
  
  monthlyAnalysis: {
    title: 'Monthly Analysis',
    table: {
      month: 'Month',
      revenue: 'Revenue'
    }
  },
  
  summary: {
    totalRevenueYTD: 'Total Revenue (YTD)',
    avgMonthlyRevenue: 'Avg Monthly Revenue'
  }
}
```

### Keys Naming Convention:
- **camelCase** for all keys
- **Descriptive names** (quarter, not q)
- **Hierarchical** (feature → section → item)
- **Singular/plural** as needed (use consistent patterns)

## Expanding to More Languages

### Add German Support:

1. **Create de.js:**
```javascript
// src/locales/de.js
export default {
  reports: {
    title: 'Leistungsberichte',
    description: 'Quartalsleistungsmetriken anzeigen',
    // ... all keys
  }
}
```

2. **Update useI18n.js:**
```javascript
import en from '../locales/en'
import ja from '../locales/ja'
import de from '../locales/de'  // Add this

const translations = { en, ja, de }
```

3. **Update language names:**
```javascript
const names = {
  en: 'English',
  ja: '日本語',
  de: 'Deutsch'  // Add this
}
```

## Debugging Tips

### Check Translation Key:
```javascript
const { t } = useI18n()

console.log(t('reports.title'))  // Should show translated string
console.log(t('invalid.key'))     // Shows the key itself (fallback)
```

### Verify Language Setting:
```javascript
const { currentLocale, availableLocales } = useI18n()

console.log('Current:', currentLocale.value)      // 'en' or 'ja'
console.log('Available:', availableLocales.value)  // ['en', 'ja']
console.log('Storage:', localStorage.getItem('app-locale'))  // Should match
```

### Test Fallback:
```javascript
// Temporarily remove a translation to test fallback
const { t, setLocale } = useI18n()

setLocale('ja')
console.log(t('reports.title'))  // Should show Japanese
// If removed from ja.js, will show English "Performance Reports"
```

## Performance Tips

1. **Use computed properties for translations**
   - Avoid recalculating in render
   ```javascript
   const title = computed(() => t('reports.title'))
   ```

2. **Watch language changes efficiently**
   - Only re-render if needed
   ```javascript
   watch(currentLocale, () => {
     // Only update if component has dynamic content
   })
   ```

3. **Memoize translation lookups**
   - Cache complex translation operations
   ```javascript
   const memoizedTranslation = computed(() => {
     return t('complex.nested.key')
   })
   ```

## Common Issues & Solutions

| Issue | Solution |
|-------|----------|
| Text not updating after language change | Watch `currentLocale` and trigger re-render |
| Missing translation key shows as `'key'` | Add key to all language files or English fallback |
| LocalStorage not saving preference | Check browser privacy settings |
| Special characters corrupted | Save files as UTF-8 encoding |
| Layout breaks with long translations | Use flexible CSS (flex, grid) that adapts |

## Testing Your i18n

### Manual Testing Checklist:
- [ ] Switch language and verify all text changes
- [ ] Refresh page and verify language preference persists
- [ ] Check all table headers are translated
- [ ] Verify dynamic content (cards, lists) shows correct language
- [ ] Test missing translation fallback to English
- [ ] Verify special characters display correctly
- [ ] Test on mobile to ensure layout adapts to translated text length

### Automated Test Example:
```javascript
import { useI18n } from '@/composables/useI18n'

describe('Internationalization', () => {
  it('translates reports title', () => {
    const { t, setLocale } = useI18n()
    
    setLocale('en')
    expect(t('reports.title')).toBe('Performance Reports')
    
    setLocale('ja')
    expect(t('reports.title')).toBe('パフォーマンスレポート')
  })

  it('persists language preference', () => {
    const { setLocale } = useI18n()
    setLocale('ja')
    expect(localStorage.getItem('app-locale')).toBe('ja')
  })
})
```

## Next Steps

1. **Copy composables/translation files to your project**
2. **Add language switcher to your header**
3. **Identify all user-visible strings in your components**
4. **Replace hardcoded strings with `t()` calls**
5. **Test language switching**
6. **Add more languages as needed**

## Additional Resources

- **Full Documentation**: See `i18n-agent-documentation.md`
- **Vue 3 Composition API**: https://v3.vuejs.org/guide/composition-api-introduction.html
- **Vue 3 Reactivity**: https://v3.vuejs.org/guide/reactivity.html
- **Best Practices**: Follow the patterns in this quickstart

---

**Happy translating! 🌍**
