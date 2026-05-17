<template>
  <div class="restocking-page">
    <div class="page-header">
      <h1>{{ t('restocking.title') }}</h1>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div class="settings-row">
      <div class="budget-control">
        <label>{{ t('restocking.budget') }}</label>
        <div class="slider-row">
          <input
            type="range"
            v-model.number="budget"
            min="1000"
            max="100000"
            step="500"
            class="budget-slider"
          />
          <input
            type="number"
            v-model.number="budget"
            min="0"
            class="budget-input"
          />
        </div>
      </div>
      <div class="lead-time-control">
        <label>{{ t('restocking.leadTime') }}</label>
        <select v-model.number="leadTimeDays" class="lead-time-select">
          <option :value="7">7 {{ t('restocking.days') }}</option>
          <option :value="14">14 {{ t('restocking.days') }}</option>
          <option :value="30">30 {{ t('restocking.days') }}</option>
        </select>
      </div>
    </div>

    <div class="card budget-summary">
      <div class="summary-col">
        <div class="label">{{ t('restocking.selectedTotal') }}</div>
        <div class="value">${{ selectedTotal.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</div>
      </div>
      <div class="summary-col progress-col">
        <div class="budget-bar">
          <div
            class="budget-fill"
            :style="{
              width: budgetPercent + '%',
              background: remaining < 0 ? '#ef4444' : '#3b82f6'
            }"
          ></div>
        </div>
        <div class="budget-label">{{ (budgetPercent).toFixed(0) }}%</div>
      </div>
      <div class="summary-col">
        <div class="label">{{ t('restocking.remaining') }}</div>
        <div class="value" :class="{ 'text-red': remaining < 0 }">
          ${{ remaining.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}
        </div>
      </div>
    </div>

    <div v-if="loading" class="card">
      <p class="text-center">{{ t('common.loading') }}</p>
    </div>
    <div v-else-if="error" class="card error-card">
      <p>{{ error }}</p>
    </div>
    <div v-else class="card checklist-card">
      <h3 class="card-title">{{ t('restocking.checklist') }}</h3>
      <div class="table-container">
        <table class="restock-table">
          <thead>
            <tr>
              <th class="col-check">
                <input
                  type="checkbox"
                  v-model="selectAll"
                  class="select-all-checkbox"
                />
              </th>
              <th class="col-item">{{ t('restocking.table.item') }}</th>
              <th class="col-sku">{{ t('restocking.table.sku') }}</th>
              <th class="col-trend">{{ t('restocking.table.trend') }}</th>
              <th class="col-num">{{ t('restocking.table.current') }}</th>
              <th class="col-num">{{ t('restocking.table.forecasted') }}</th>
              <th class="col-num">{{ t('restocking.table.gap') }}</th>
              <th class="col-cost">{{ t('restocking.table.unitCost') }}</th>
              <th class="col-qty">{{ t('restocking.table.qty') }}</th>
              <th class="col-total">{{ t('restocking.table.lineTotal') }}</th>
            </tr>
          </thead>
          <tbody>
            <tr
              v-for="forecast in forecasts"
              :key="forecast.id"
              :class="{
                'row-checked': sel[forecast.id]?.checked,
                'row-disabled': !forecast.unit_cost
              }"
            >
              <td class="col-check">
                <input
                  type="checkbox"
                  v-model="sel[forecast.id].checked"
                  :disabled="!forecast.unit_cost"
                  class="item-checkbox"
                />
              </td>
              <td class="col-item">{{ forecast.item_name }}</td>
              <td class="col-sku">{{ forecast.item_sku }}</td>
              <td class="col-trend">
                <span
                  :class="[
                    'trend-badge',
                    forecast.trend === 'increasing'
                      ? 'badge-increasing'
                      : forecast.trend === 'decreasing'
                      ? 'badge-decreasing'
                      : 'badge-stable'
                  ]"
                >
                  {{ forecast.trend }}
                </span>
              </td>
              <td class="col-num">{{ forecast.current_demand }}</td>
              <td class="col-num">{{ forecast.forecasted_demand }}</td>
              <td class="col-num">{{ Math.max(0, forecast.forecasted_demand - forecast.current_demand) }}</td>
              <td class="col-cost">
                {{
                  forecast.unit_cost
                    ? '$' + forecast.unit_cost.toFixed(2)
                    : t('restocking.noUnitCost')
                }}
              </td>
              <td class="col-qty">
                <input
                  type="number"
                  v-model.number="sel[forecast.id].qty"
                  min="0"
                  :disabled="!sel[forecast.id].checked"
                  class="qty-input"
                />
              </td>
              <td class="col-total">
                {{
                  sel[forecast.id].checked && forecast.unit_cost
                    ? '$' +
                      (sel[forecast.id].qty * forecast.unit_cost).toLocaleString(
                        'en-US',
                        { minimumFractionDigits: 2, maximumFractionDigits: 2 }
                      )
                    : '—'
                }}
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div class="action-row">
      <div v-if="!loading && !error" class="validation-messages">
        <div v-if="checkedItems.length === 0 && !submitting" class="message-info">
          {{ t('restocking.nothingSelected') }}
        </div>
        <div v-if="remaining < 0" class="message-error">
          {{ t('restocking.overBudget') }}
        </div>
      </div>
      <button
        class="btn-primary btn-place-order"
        @click="placeOrder"
        :disabled="!canPlaceOrder"
      >
        {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
      </button>
    </div>

    <Transition name="fade">
      <div v-if="orderSuccess" class="success-message">
        {{ t('restocking.orderSuccess') }}
      </div>
    </Transition>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()

    // Data loading
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])

    // Settings
    const budget = ref(10000)
    const leadTimeDays = ref(14)

    // Checklist state
    const sel = ref({})

    // Submission state
    const submitting = ref(false)
    const orderSuccess = ref(false)

    // Computed properties
    const checkedItems = computed(() =>
      forecasts.value.filter(f => sel.value[f.id]?.checked)
    )

    const selectedTotal = computed(() =>
      checkedItems.value.reduce((sum, f) => {
        const qty = sel.value[f.id]?.qty ?? Math.max(0, f.forecasted_demand - f.current_demand)
        const cost = f.unit_cost ?? 0
        return sum + qty * cost
      }, 0)
    )

    const remaining = computed(() => budget.value - selectedTotal.value)

    const budgetPercent = computed(() =>
      Math.min(100, (selectedTotal.value / Math.max(1, budget.value)) * 100)
    )

    const canPlaceOrder = computed(() =>
      checkedItems.value.length > 0 && remaining.value >= 0 && !submitting.value
    )

    const expectedDelivery = computed(() => {
      const d = new Date()
      d.setDate(d.getDate() + Number(leadTimeDays.value))
      return d.toISOString().split('T')[0]
    })

    const selectAll = computed({
      get: () =>
        forecasts.value
          .filter(f => f.unit_cost)
          .every(f => sel.value[f.id]?.checked),
      set: (val) =>
        forecasts.value
          .filter(f => f.unit_cost)
          .forEach(f => {
            sel.value[f.id].checked = val
          })
    })

    // Methods
    const loadForecasts = async () => {
      try {
        loading.value = true
        error.value = null
        const data = await api.getDemandForecasts()
        forecasts.value = data
        const initial = {}
        data.forEach(f => {
          const gap = f.forecasted_demand - f.current_demand
          initial[f.id] = {
            checked: false,
            qty: Math.max(0, gap)
          }
        })
        sel.value = initial
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (!canPlaceOrder.value) return
      submitting.value = true
      try {
        const now = new Date().toISOString()
        const items = checkedItems.value.map(f => ({
          sku: f.item_sku,
          name: f.item_name,
          quantity: sel.value[f.id].qty,
          unit_price: f.unit_cost ?? 0
        }))
        await api.createOrder({
          customer: 'Internal Restock',
          items,
          order_date: now,
          expected_delivery: expectedDelivery.value,
          total_value: selectedTotal.value,
          warehouse: null,
          category: null
        })
        orderSuccess.value = true
        Object.keys(sel.value).forEach(id => {
          sel.value[id].checked = false
        })
        setTimeout(() => {
          orderSuccess.value = false
        }, 5000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      loading,
      error,
      forecasts,
      budget,
      leadTimeDays,
      sel,
      submitting,
      orderSuccess,
      checkedItems,
      selectedTotal,
      remaining,
      budgetPercent,
      canPlaceOrder,
      expectedDelivery,
      selectAll,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking-page {
  padding: 2rem;
  max-width: 1600px;
  margin: 0 auto;
}

.page-header {
  margin-bottom: 2rem;
}

.page-header h1 {
  font-size: 2rem;
  color: #0f172a;
  margin-bottom: 0.5rem;
}

.page-header p {
  color: #64748b;
  font-size: 1rem;
}

.settings-row {
  display: flex;
  gap: 2rem;
  align-items: flex-end;
  margin-bottom: 1.5rem;
  flex-wrap: wrap;
}

.budget-control,
.lead-time-control {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
  flex: 0 0 auto;
}

.budget-control label,
.lead-time-control label {
  font-weight: 600;
  color: #0f172a;
  font-size: 0.95rem;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
}

.budget-slider {
  flex: 1;
  min-width: 200px;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  -webkit-appearance: none;
  appearance: none;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  box-shadow: 0 2px 4px rgba(59, 130, 246, 0.3);
}

.budget-slider::-moz-range-thumb {
  width: 18px;
  height: 18px;
  border-radius: 50%;
  background: #3b82f6;
  cursor: pointer;
  border: none;
  box-shadow: 0 2px 4px rgba(59, 130, 246, 0.3);
}

.budget-input {
  width: 120px;
  padding: 0.5rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.95rem;
  color: #0f172a;
}

.budget-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.lead-time-select {
  padding: 0.5rem 0.75rem;
  border: 1px solid #cbd5e1;
  border-radius: 6px;
  font-size: 0.95rem;
  color: #0f172a;
  background: white;
  cursor: pointer;
}

.lead-time-select:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
}

.card {
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 1.5rem;
  margin-bottom: 1.5rem;
}

.budget-summary {
  display: grid;
  grid-template-columns: 1fr 2fr 1fr;
  gap: 1.5rem;
  align-items: center;
  background: linear-gradient(135deg, #f8fafc 0%, #f0f4f8 100%);
}

.summary-col {
  display: flex;
  flex-direction: column;
  gap: 0.5rem;
}

.summary-col .label {
  font-weight: 500;
  color: #64748b;
  font-size: 0.9rem;
}

.summary-col .value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.summary-col .value.text-red {
  color: #ef4444;
}

.progress-col {
  flex-direction: row;
  align-items: center;
  gap: 1rem;
}

.budget-bar {
  flex: 1;
  height: 8px;
  background: #e2e8f0;
  border-radius: 9999px;
  overflow: hidden;
}

.budget-fill {
  height: 100%;
  border-radius: 9999px;
  transition: width 0.3s ease, background 0.3s ease;
}

.budget-label {
  font-weight: 600;
  color: #0f172a;
  min-width: 45px;
  text-align: right;
}

.error-card {
  background: #fee2e2;
  border-color: #fca5a5;
}

.error-card p {
  color: #991b1b;
  margin: 0;
}

.text-center {
  text-align: center;
  color: #64748b;
}

.checklist-card {
  overflow: hidden;
}

.card-title {
  font-size: 1.25rem;
  color: #0f172a;
  margin-bottom: 1.5rem;
  font-weight: 600;
}

.table-container {
  overflow-x: auto;
}

.restock-table {
  width: 100%;
  border-collapse: collapse;
  table-layout: fixed;
}

.restock-table thead {
  background: #f8fafc;
  border-bottom: 2px solid #e2e8f0;
}

.restock-table th {
  padding: 1rem;
  text-align: left;
  font-weight: 600;
  color: #0f172a;
  font-size: 0.9rem;
  white-space: nowrap;
}

.restock-table td {
  padding: 1rem;
  border-bottom: 1px solid #e2e8f0;
  color: #475569;
  font-size: 0.95rem;
}

.restock-table tbody tr {
  transition: background 0.2s ease;
}

.restock-table tbody tr:hover {
  background: #f8fafc;
}

.restock-table tbody tr.row-checked {
  background: #eff6ff;
}

.restock-table tbody tr.row-disabled {
  opacity: 0.5;
  pointer-events: none;
}

.col-check {
  width: 36px;
}

.col-item {
  width: 180px;
}

.col-sku {
  width: 90px;
}

.col-trend {
  width: 90px;
}

.col-num {
  width: 80px;
  text-align: right;
}

.col-cost {
  width: 90px;
  text-align: right;
}

.col-qty {
  width: 75px;
}

.col-total {
  width: 100px;
  text-align: right;
}

.select-all-checkbox,
.item-checkbox {
  cursor: pointer;
  width: 18px;
  height: 18px;
  accent-color: #3b82f6;
}

.select-all-checkbox:disabled,
.item-checkbox:disabled {
  cursor: not-allowed;
  opacity: 0.5;
}

.trend-badge {
  display: inline-block;
  padding: 4px 8px;
  border-radius: 4px;
  font-size: 0.85rem;
  font-weight: 500;
}

.badge-increasing {
  background: #dcfce7;
  color: #166534;
}

.badge-decreasing {
  background: #fee2e2;
  color: #991b1b;
}

.badge-stable {
  background: #f3e8ff;
  color: #6b21a8;
}

.qty-input {
  width: 100%;
  padding: 0.4rem;
  border: 1px solid #cbd5e1;
  border-radius: 4px;
  font-size: 0.9rem;
  text-align: right;
}

.qty-input:focus {
  outline: none;
  border-color: #3b82f6;
  box-shadow: 0 0 0 2px rgba(59, 130, 246, 0.1);
}

.qty-input:disabled {
  background: #f8fafc;
  cursor: not-allowed;
  opacity: 0.5;
}

.action-row {
  display: flex;
  align-items: center;
  gap: 2rem;
  justify-content: space-between;
  margin-top: 2rem;
}

.validation-messages {
  flex: 1;
  display: flex;
  gap: 1rem;
  flex-wrap: wrap;
}

.message-info,
.message-error {
  padding: 0.75rem 1rem;
  border-radius: 6px;
  font-size: 0.9rem;
  font-weight: 500;
}

.message-info {
  background: #dbeafe;
  color: #0369a1;
}

.message-error {
  background: #fee2e2;
  color: #991b1b;
}

.btn-place-order {
  padding: 0.75rem 2rem;
  background: #3b82f6;
  color: white;
  border: none;
  border-radius: 6px;
  font-weight: 600;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.2s ease;
  white-space: nowrap;
}

.btn-place-order:hover:not(:disabled) {
  background: #2563eb;
  box-shadow: 0 4px 12px rgba(59, 130, 246, 0.3);
}

.btn-place-order:disabled {
  background: #cbd5e1;
  cursor: not-allowed;
  opacity: 0.7;
}

.success-message {
  background: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 1rem;
  margin-top: 1rem;
  text-align: center;
  font-weight: 500;
}

.fade-enter-active,
.fade-leave-active {
  transition: opacity 0.3s ease;
}

.fade-enter-from,
.fade-leave-to {
  opacity: 0;
}

@media (max-width: 768px) {
  .restocking-page {
    padding: 1rem;
  }

  .settings-row {
    flex-direction: column;
    gap: 1rem;
  }

  .budget-control,
  .lead-time-control {
    flex: 1;
    width: 100%;
  }

  .budget-summary {
    grid-template-columns: 1fr;
    gap: 1rem;
  }

  .slider-row {
    flex-direction: column;
  }

  .budget-slider {
    min-width: 100%;
  }

  .budget-input {
    width: 100%;
  }

  .action-row {
    flex-direction: column;
    gap: 1rem;
  }

  .btn-place-order {
    width: 100%;
  }

  .col-item {
    width: 130px;
  }

  .col-sku {
    width: 70px;
  }

  .col-trend {
    width: 70px;
  }

  .col-num {
    width: 65px;
  }

  .col-cost {
    width: 75px;
  }

  .col-qty {
    width: 65px;
  }

  .col-total {
    width: 85px;
  }
}
</style>
