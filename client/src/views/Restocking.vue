<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Recommend items to restock based on demand forecasts and available budget</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Success Banner -->
      <div v-if="successMessage" class="success-banner">
        <div class="success-banner-content">
          <span>{{ successMessage }}</span>
          <router-link to="/orders" class="view-orders-link">View in Orders</router-link>
        </div>
      </div>

      <!-- Budget Section -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
        </div>
        <div class="budget-controls">
          <div class="budget-display">
            <span class="budget-value">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
          </div>
          <input
            type="range"
            :min="50000"
            :max="2000000"
            :step="10000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="budget-range-labels">
            <span>{{ currencySymbol }}50,000</span>
            <span>{{ currencySymbol }}2,000,000</span>
          </div>
          <div class="budget-summary">
            <span class="summary-item">
              <strong>{{ selectedItems.length }}</strong> items selected
            </span>
            <span class="summary-separator">·</span>
            <span class="summary-item">
              Total cost: <strong>{{ currencySymbol }}{{ totalSelectedCost.toLocaleString() }}</strong>
            </span>
            <span class="summary-separator">·</span>
            <span class="summary-item" :class="{ 'remaining-ok': remainingBudget >= 0, 'remaining-over': remainingBudget < 0 }">
              Remaining: <strong>{{ currencySymbol }}{{ remainingBudget.toLocaleString() }}</strong>
            </span>
          </div>
        </div>
      </div>

      <!-- Recommendations Table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items</h3>
          <span class="badge info">{{ selectedItems.length }} / {{ recommendations.length }}</span>
        </div>
        <div class="table-container">
          <table class="recommendations-table">
            <thead>
              <tr>
                <th class="col-item">Item</th>
                <th class="col-trend">Trend</th>
                <th class="col-demand">Current Demand</th>
                <th class="col-demand">Forecasted Demand</th>
                <th class="col-qty">Qty to Order</th>
                <th class="col-cost">Unit Cost</th>
                <th class="col-cost">Est. Cost</th>
                <th class="col-budget">In Budget</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.item_sku"
                :class="{ 'row-over-budget': !isInBudget(item) }"
              >
                <td class="col-item">
                  <div class="item-name">{{ item.item_name }}</div>
                  <div class="item-sku">{{ item.item_sku }}</div>
                </td>
                <td class="col-trend">
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td class="col-demand">{{ item.current_demand.toLocaleString() }}</td>
                <td class="col-demand">{{ item.forecasted_demand.toLocaleString() }}</td>
                <td class="col-qty">{{ item.quantity_to_order.toLocaleString() }}</td>
                <td class="col-cost">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td class="col-cost"><strong>{{ currencySymbol }}{{ item.total_cost.toLocaleString() }}</strong></td>
                <td class="col-budget">
                  <span v-if="isInBudget(item)" class="in-budget-indicator in-budget">
                    <svg width="16" height="16" viewBox="0 0 16 16" fill="none" aria-hidden="true">
                      <circle cx="8" cy="8" r="7" fill="#d1fae5" stroke="#059669" stroke-width="1.5"/>
                      <path d="M5 8l2 2 4-4" stroke="#059669" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/>
                    </svg>
                    <span class="indicator-label">In Budget</span>
                  </span>
                  <span v-else class="in-budget-indicator over-budget">
                    <svg width="16" height="16" viewBox="0 0 16 16" fill="none" aria-hidden="true">
                      <circle cx="8" cy="8" r="7" fill="#f1f5f9" stroke="#94a3b8" stroke-width="1.5"/>
                      <path d="M8 5v3M8 10v1" stroke="#94a3b8" stroke-width="1.5" stroke-linecap="round"/>
                    </svg>
                    <span class="indicator-label">Over Budget</span>
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order Button -->
      <div class="order-actions">
        <button
          class="place-order-btn"
          :disabled="selectedItems.length === 0 || submitting || submitted"
          @click="placeOrder"
        >
          <span v-if="submitting">Submitting...</span>
          <span v-else>
            Place Restocking Order ({{ selectedItems.length }} items · {{ currencySymbol }}{{ totalSelectedCost.toLocaleString() }})
          </span>
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { currentCurrency } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const submitted = ref(false)
    const successMessage = ref('')
    const recommendations = ref([])
    const budget = ref(500000)

    // Set of item_skus that are within budget (greedy selection)
    const selectedItems = computed(() => {
      let remaining = budget.value
      const result = []
      for (const item of recommendations.value) {
        if (item.total_cost <= remaining) {
          result.push(item)
          remaining -= item.total_cost
        }
      }
      return result
    })

    const selectedSkuSet = computed(() => {
      return new Set(selectedItems.value.map(i => i.item_sku))
    })

    const isInBudget = (item) => {
      return selectedSkuSet.value.has(item.item_sku)
    }

    const totalSelectedCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.total_cost, 0)
    })

    const remainingBudget = computed(() => {
      return budget.value - totalSelectedCost.value
    })

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const data = await api.getRestockingRecommendations()
        recommendations.value = data
      } catch (err) {
        error.value = 'Failed to load restocking recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      error.value = null
      try {
        const items = selectedItems.value.map(item => ({
          item_sku: item.item_sku,
          item_name: item.item_name,
          quantity: item.quantity_to_order,
          unit_cost: item.unit_cost,
          total_cost: item.total_cost
        }))
        const result = await api.submitRestockingOrder(items, budget.value)
        submitted.value = true
        successMessage.value = `Order ${result.order_number} submitted successfully. Expected delivery in ${result.lead_time_days} days.`
        // Reset after 5 seconds
        setTimeout(() => {
          submitted.value = false
          successMessage.value = ''
        }, 5000)
      } catch (err) {
        error.value = 'Failed to submit restocking order'
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadRecommendations())

    return {
      loading,
      error,
      submitting,
      submitted,
      successMessage,
      recommendations,
      budget,
      currencySymbol,
      selectedItems,
      totalSelectedCost,
      remainingBudget,
      isInBudget,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  /* uses global .page-header, .card, etc. */
}

.budget-card .card-header {
  margin-bottom: 0;
  border-bottom: none;
  padding-bottom: 0;
}

.budget-controls {
  display: flex;
  flex-direction: column;
  gap: 0.75rem;
  padding-top: 0.875rem;
}

.budget-display {
  display: flex;
  align-items: baseline;
  gap: 0.5rem;
}

.budget-value {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  -webkit-appearance: none;
  appearance: none;
  background: #e2e8f0;
  border-radius: 3px;
  outline: none;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  background: #2563eb;
  border-radius: 50%;
  cursor: pointer;
  border: 2px solid white;
  box-shadow: 0 1px 4px rgba(0, 0, 0, 0.2);
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
}

.budget-summary {
  display: flex;
  align-items: center;
  gap: 0.5rem;
  flex-wrap: wrap;
  font-size: 0.875rem;
  color: #475569;
  background: #f8fafc;
  border: 1px solid #e2e8f0;
  border-radius: 8px;
  padding: 0.625rem 1rem;
}

.summary-separator {
  color: #cbd5e1;
  font-size: 1rem;
}

.summary-item strong {
  color: #0f172a;
}

.remaining-ok strong {
  color: #059669;
}

.remaining-over strong {
  color: #dc2626;
}

.recommendations-table {
  table-layout: fixed;
  width: 100%;
}

.col-item {
  width: 220px;
}

.col-trend {
  width: 110px;
}

.col-demand {
  width: 130px;
}

.col-qty {
  width: 110px;
}

.col-cost {
  width: 110px;
}

.col-budget {
  width: 120px;
}

.item-name {
  font-size: 0.875rem;
  font-weight: 500;
  color: #0f172a;
}

.item-sku {
  font-size: 0.75rem;
  color: #94a3b8;
  margin-top: 2px;
}

.row-over-budget {
  opacity: 0.5;
}

.in-budget-indicator {
  display: inline-flex;
  align-items: center;
  gap: 0.375rem;
  font-size: 0.75rem;
  font-weight: 600;
}

.in-budget .indicator-label {
  color: #059669;
}

.over-budget .indicator-label {
  color: #94a3b8;
}

.order-actions {
  display: flex;
  justify-content: flex-end;
  margin-top: 0.5rem;
  margin-bottom: 1.5rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 1.75rem;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  background: #94a3b8;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  margin-bottom: 1.25rem;
}

.success-banner-content {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 1rem;
  font-size: 0.938rem;
  color: #065f46;
  font-weight: 500;
}

.view-orders-link {
  color: #059669;
  font-weight: 600;
  text-decoration: underline;
  white-space: nowrap;
}

.view-orders-link:hover {
  color: #047857;
}
</style>
