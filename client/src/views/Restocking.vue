<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Set a budget and get automated restock recommendations based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading demand forecasts...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Slider Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-control">
          <div class="budget-display">{{ formatCurrency(budget) }}</div>
          <input
            type="range"
            class="budget-slider"
            min="1000"
            max="100000"
            step="1000"
            v-model.number="budget"
          />
          <div class="budget-range-labels">
            <span>$1,000</span>
            <span>$100,000</span>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations</h3>
          <div class="budget-summary">
            <span class="remaining-label">Remaining budget:</span>
            <span class="remaining-value" :class="{ 'remaining-low': remainingBudget < budget * 0.1 }">
              {{ formatCurrency(remainingBudget) }}
            </span>
          </div>
        </div>

        <div v-if="recommendations.length === 0" class="empty-state">
          No items fit within the current budget. Try increasing the budget.
        </div>
        <div v-else>
          <div class="table-container">
            <table class="restock-table">
              <thead>
                <tr>
                  <th>Item Name</th>
                  <th>SKU</th>
                  <th>Trend</th>
                  <th class="col-num">Forecasted Demand</th>
                  <th class="col-num">Unit Price</th>
                  <th class="col-num">Qty to Order</th>
                  <th class="col-num">Subtotal</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendations" :key="item.item_sku">
                  <td>{{ item.item_name }}</td>
                  <td><code class="sku">{{ item.item_sku }}</code></td>
                  <td>
                    <span :class="['badge', getTrendClass(item.trend)]">{{ item.trend }}</span>
                  </td>
                  <td class="col-num">{{ item.forecasted_demand.toLocaleString() }}</td>
                  <td class="col-num">{{ formatCurrency(item.unit_price) }}</td>
                  <td class="col-num">{{ item.quantity.toLocaleString() }}</td>
                  <td class="col-num"><strong>{{ formatCurrency(item.subtotal) }}</strong></td>
                </tr>
              </tbody>
              <tfoot>
                <tr class="total-row">
                  <td colspan="6" class="total-label">Total</td>
                  <td class="col-num total-value">
                    <strong>{{ formatCurrency(totalCost) }}</strong>
                  </td>
                </tr>
              </tfoot>
            </table>
          </div>

          <!-- Order Button & Success Message -->
          <div class="order-actions">
            <div v-if="orderSuccess" class="success-message">
              {{ orderSuccess }}
            </div>
            <button
              class="btn-place-order"
              :disabled="recommendations.length === 0 || submitting"
              @click="placeOrder"
            >
              {{ submitting ? 'Placing Order...' : 'Place Restock Order' }}
            </button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(20000)
    const submitting = ref(false)
    const orderSuccess = ref(null)

    // Load demand data with prices on mount
    const loadForecasts = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getDemandWithPrices()
        forecasts.value = data
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Greedy recommendation algorithm:
    // 1. Sort: "increasing" trend first, then by forecasted_demand descending
    // 2. Pick items using full forecasted_demand qty; skip if cost exceeds remaining budget
    const recommendations = computed(() => {
      const sorted = [...forecasts.value].sort((a, b) => {
        if (a.trend === 'increasing' && b.trend !== 'increasing') return -1
        if (b.trend === 'increasing' && a.trend !== 'increasing') return 1
        return b.forecasted_demand - a.forecasted_demand
      })

      let remaining = budget.value
      const picked = []

      for (const forecast of sorted) {
        const qty = forecast.forecasted_demand
        const cost = qty * forecast.unit_price
        if (cost <= remaining) {
          picked.push({
            ...forecast,
            quantity: qty,
            subtotal: cost
          })
          remaining -= cost
        }
      }

      return picked
    })

    const totalCost = computed(() =>
      recommendations.value.reduce((sum, item) => sum + item.subtotal, 0)
    )

    const remainingBudget = computed(() => budget.value - totalCost.value)

    // Place the restock order via API
    const placeOrder = async () => {
      if (recommendations.value.length === 0 || submitting.value) return

      submitting.value = true
      orderSuccess.value = null
      try {
        const items = recommendations.value.map(item => ({
          sku: item.item_sku,
          name: item.item_name,
          quantity: item.quantity,
          unit_price: item.unit_price
        }))
        const result = await api.submitRestockOrder(items)
        orderSuccess.value = `Order ${result.order_number} placed! Expected delivery in ${result.lead_days} days.`

        // Clear success message after 5 seconds
        setTimeout(() => {
          orderSuccess.value = null
        }, 5000)
      } catch (err) {
        error.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    const formatCurrency = (value) => {
      return '$' + value.toLocaleString('en-US', { minimumFractionDigits: 2, maximumFractionDigits: 2 })
    }

    const getTrendClass = (trend) => {
      const map = {
        'increasing': 'success',
        'decreasing': 'danger',
        'stable': 'info'
      }
      return map[trend] || 'info'
    }

    onMounted(() => loadForecasts())

    return {
      loading,
      error,
      budget,
      submitting,
      orderSuccess,
      recommendations,
      totalCost,
      remainingBudget,
      placeOrder,
      formatCurrency,
      getTrendClass
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 2rem;
}

/* Budget card */
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-control {
  padding: 1rem 1.5rem 1.5rem;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  margin-bottom: 1rem;
}

.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  margin-top: 0.5rem;
  font-size: 0.813rem;
  color: #64748b;
}

/* Budget summary in card header */
.budget-summary {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  font-size: 0.875rem;
}

.remaining-label {
  color: #64748b;
}

.remaining-value {
  font-weight: 600;
  color: #059669;
}

.remaining-value.remaining-low {
  color: #ea580c;
}

/* Table */
.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.col-num {
  text-align: right;
}

/* Total row */
.total-row {
  background: #f8fafc;
  border-top: 2px solid #e2e8f0;
}

.total-label {
  text-align: right;
  font-weight: 600;
  color: #0f172a;
  padding-right: 1rem;
}

.total-value {
  font-size: 1rem;
  color: #0f172a;
}

/* SKU code style */
.sku {
  font-size: 0.813rem;
  background: #f1f5f9;
  padding: 0.125rem 0.375rem;
  border-radius: 4px;
  color: #475569;
  font-family: monospace;
}

/* Empty state */
.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  font-size: 0.938rem;
}

/* Order action bar */
.order-actions {
  display: flex;
  align-items: center;
  gap: 1rem;
  padding: 1rem 1.5rem;
  border-top: 1px solid #e2e8f0;
  justify-content: flex-end;
}

.success-message {
  font-size: 0.875rem;
  font-weight: 500;
  color: #059669;
  background: #f0fdf4;
  border: 1px solid #bbf7d0;
  border-radius: 6px;
  padding: 0.5rem 0.875rem;
}

.btn-place-order {
  background: #2563eb;
  color: #fff;
  border: none;
  border-radius: 6px;
  padding: 0.625rem 1.5rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}
</style>
