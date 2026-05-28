<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your budget and we'll recommend items to restock based on forecasted demand.</p>
    </div>

    <div v-if="loadError" class="error">{{ loadError }}</div>

    <div class="restocking-grid">
      <!-- LEFT: Budget panel -->
      <div class="card budget-panel">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>

        <div class="budget-amount">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>

        <input
          type="range"
          class="slider"
          :min="0"
          :max="BUDGET_MAX"
          :step="1000"
          v-model.number="budget"
        />
        <div class="slider-range">
          <span>{{ currencySymbol }}0</span>
          <span>{{ currencySymbol }}{{ BUDGET_MAX.toLocaleString() }}</span>
        </div>

        <div class="budget-summary">
          <div class="summary-row">
            <span class="summary-label">Items selected</span>
            <span class="summary-value">{{ itemsCount }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">Total cost</span>
            <span class="summary-value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</span>
          </div>
          <div class="summary-row">
            <span class="summary-label">Remaining budget</span>
            <span class="summary-value" :class="{ 'remaining-positive': remainingBudget > 0 }">
              {{ currencySymbol }}{{ remainingBudget.toLocaleString() }}
            </span>
          </div>
        </div>

        <button
          class="btn-primary"
          :disabled="itemsCount === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? 'Submitting...' : 'Place Order' }}
        </button>

        <div v-if="submittedMessage" class="success-message">{{ submittedMessage }}</div>
        <div v-if="errorMessage" class="error">{{ errorMessage }}</div>
      </div>

      <!-- RIGHT: Recommended items -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
        </div>

        <div v-if="forecasts.length === 0 && !loadError" class="loading">Loading forecasts...</div>

        <div v-else-if="recommendations.length === 0" class="empty-state">
          Increase your budget to see recommendations.
        </div>

        <div v-else class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>SKU</th>
                <th>Item</th>
                <th>Category</th>
                <th>Forecast Demand</th>
                <th>Unit Cost</th>
                <th>Qty to Order</th>
                <th>Line Total</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td class="sku-cell">{{ item.item_sku }}</td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.category }}</td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>{{ item.quantity.toLocaleString() }}</strong></td>
                <td><strong>{{ currencySymbol }}{{ item.line_total.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
            <tfoot>
              <tr class="totals-row">
                <td colspan="5"></td>
                <td><strong>{{ totalQty.toLocaleString() }}</strong></td>
                <td><strong>{{ currencySymbol }}{{ totalCost.toLocaleString() }}</strong></td>
              </tr>
            </tfoot>
          </table>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

const BUDGET_MAX = 100000

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const forecasts = ref([])
    const budget = ref(BUDGET_MAX / 2)
    const submitting = ref(false)
    const submittedMessage = ref(null)
    const errorMessage = ref(null)
    const loadError = ref(null)

    // Recommendation algorithm: greedy by forecasted_demand DESC, never exceed budget
    const recommendations = computed(() => {
      if (forecasts.value.length === 0) return []

      const sorted = [...forecasts.value].sort((a, b) => b.forecasted_demand - a.forecasted_demand)
      const result = []
      let remaining = budget.value

      for (const item of sorted) {
        if (remaining <= 0) break
        if (!item.unit_cost || item.unit_cost <= 0) continue

        // Find cheapest remaining cost to know if we can continue
        const maxAffordable = Math.floor(remaining / item.unit_cost)
        const qty = Math.min(item.forecasted_demand, maxAffordable)

        if (qty > 0) {
          const line_total = qty * item.unit_cost
          result.push({
            item_sku: item.item_sku,
            item_name: item.item_name,
            category: item.category,
            forecasted_demand: item.forecasted_demand,
            unit_cost: item.unit_cost,
            quantity: qty,
            line_total
          })
          remaining -= line_total
        }

        // Stop early if remaining is less than the cheapest un-included item cost
        const unincluded = sorted.filter(s => !result.find(r => r.item_sku === s.item_sku))
        if (unincluded.length > 0) {
          const minCost = Math.min(...unincluded.map(s => s.unit_cost).filter(c => c > 0))
          if (remaining < minCost) break
        }
      }

      return result
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.line_total, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    const itemsCount = computed(() => recommendations.value.length)

    const totalQty = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.quantity, 0)
    )

    const loadForecasts = async () => {
      loadError.value = null
      try {
        forecasts.value = await api.getDemandForecasts()
      } catch (err) {
        loadError.value = 'Failed to load demand forecasts: ' + (err.message || 'Unknown error')
        console.error(err)
      }
    }

    const placeOrder = async () => {
      if (itemsCount.value === 0 || submitting.value) return

      submitting.value = true
      submittedMessage.value = null
      errorMessage.value = null

      try {
        const payload = {
          budget: budget.value,
          items: recommendations.value.map(item => ({
            item_sku: item.item_sku,
            item_name: item.item_name,
            category: item.category,
            quantity: item.quantity,
            unit_cost: item.unit_cost,
            line_total: item.line_total
          }))
        }
        const order = await api.createRestockingOrder(payload)
        submittedMessage.value = `Order ${order.order_number} submitted — expected delivery in ${order.lead_time_days} days.`

        window.dispatchEvent(new CustomEvent('restocking-order-submitted'))

        setTimeout(() => {
          submittedMessage.value = null
        }, 4000)
      } catch (err) {
        errorMessage.value = 'Failed to submit order: ' + (err.message || 'Unknown error')
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      BUDGET_MAX,
      forecasts,
      budget,
      submitting,
      submittedMessage,
      errorMessage,
      loadError,
      recommendations,
      totalCost,
      remainingBudget,
      itemsCount,
      totalQty,
      currencySymbol,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.restocking-grid {
  display: grid;
  grid-template-columns: 340px 1fr;
  gap: 1.25rem;
  align-items: start;
}

@media (max-width: 900px) {
  .restocking-grid {
    grid-template-columns: 1fr;
  }
}

.budget-panel {
  position: sticky;
  top: 80px;
}

.budget-amount {
  font-size: 2.5rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.04em;
  margin-bottom: 1rem;
  margin-top: 0.25rem;
}

/* Slider */
.slider {
  width: 100%;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  margin-bottom: 0.375rem;
}

.slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.15s;
}

.slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 4px rgba(37, 99, 235, 0.15);
}

.slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.slider::-webkit-slider-runnable-track {
  height: 6px;
  border-radius: 3px;
}

.slider-range {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1.25rem;
}

/* Budget summary */
.budget-summary {
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.875rem;
  margin-bottom: 1.25rem;
  display: flex;
  flex-direction: column;
  gap: 0.625rem;
}

.summary-row {
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.summary-label {
  font-size: 0.875rem;
  color: #64748b;
}

.summary-value {
  font-size: 0.875rem;
  font-weight: 600;
  color: #334155;
}

.remaining-positive {
  color: #059669;
}

/* Place order button */
.btn-primary {
  width: 100%;
  padding: 0.75rem 1.25rem;
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s, opacity 0.15s;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

/* Success message */
.success-message {
  margin-top: 0.875rem;
  padding: 0.75rem 1rem;
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  color: #065f46;
  font-size: 0.875rem;
  font-weight: 500;
}

/* Table */
.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.sku-cell {
  font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
  font-size: 0.813rem;
  color: #64748b;
}

tfoot .totals-row td {
  border-top: 2px solid #e2e8f0;
  background: #f8fafc;
  font-size: 0.875rem;
  padding: 0.625rem 0.75rem;
}

/* Empty state */
.empty-state {
  text-align: center;
  padding: 3rem 1.5rem;
  color: #94a3b8;
  font-size: 0.938rem;
}
</style>
