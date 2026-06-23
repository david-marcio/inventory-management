<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set a budget to get recommendations for items that need restocking based on demand forecasts.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget Card -->
      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">Available Budget</h3>
          <span class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</span>
        </div>
        <input type="range" class="budget-slider" v-model.number="budget" :min="BUDGET_MIN" :max="BUDGET_MAX" :step="BUDGET_STEP" />
        <div class="budget-range-labels">
          <span>{{ currencySymbol }}{{ BUDGET_MIN.toLocaleString() }}</span>
          <span>{{ currencySymbol }}{{ BUDGET_MAX.toLocaleString() }}</span>
        </div>
        <!-- Utilization bar -->
        <div class="utilization-section">
          <div class="utilization-bar-track">
            <div class="utilization-bar-fill" :style="{ width: budgetUtilization + '%' }"></div>
          </div>
          <div class="utilization-labels">
            <span>Total cost: {{ currencySymbol }}{{ totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
            <span>Remaining: {{ currencySymbol }}{{ budgetRemaining.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
          </div>
        </div>
      </div>

      <!-- Recommendations Card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendedItems.length }})</h3>
          <span v-if="recommendedItems.length > 0" class="items-hint">Items below reorder point, prioritized by demand trend</span>
        </div>

        <div v-if="recommendedItems.length === 0" class="empty-state">
          <p v-if="inventoryItems.filter(i => i.quantity_on_hand < i.reorder_point).length === 0">
            All items are above their reorder point. No restocking needed.
          </p>
          <p v-else>
            No items fit within the current budget. Try increasing the budget to see recommendations.
          </p>
        </div>

        <div v-else>
          <div class="table-container">
            <table class="restocking-table">
              <thead>
                <tr>
                  <th>Item Name</th>
                  <th>SKU</th>
                  <th>Warehouse</th>
                  <th class="col-num">On Hand</th>
                  <th class="col-num">Reorder Point</th>
                  <th class="col-num">Qty to Order</th>
                  <th class="col-num">Unit Cost</th>
                  <th class="col-num">Total Cost</th>
                  <th>Demand Trend</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendedItems" :key="item.sku">
                  <td><strong>{{ item.name }}</strong></td>
                  <td class="sku-cell">{{ item.sku }}</td>
                  <td>{{ item.warehouse }}</td>
                  <td class="col-num">{{ item.quantity_on_hand }}</td>
                  <td class="col-num">{{ item.reorder_point }}</td>
                  <td class="col-num"><strong>{{ item.qty_needed }}</strong></td>
                  <td class="col-num">{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                  <td class="col-num"><strong>{{ currencySymbol }}{{ item.line_cost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</strong></td>
                  <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                </tr>
              </tbody>
            </table>
          </div>

          <!-- Order summary bar -->
          <div class="order-summary-bar">
            <div class="summary-totals">
              <div class="summary-item">
                <span class="summary-label">Total Cost</span>
                <span class="summary-value">{{ currencySymbol }}{{ totalCost.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
              </div>
              <div class="summary-item">
                <span class="summary-label">Budget Remaining</span>
                <span class="summary-value" :class="{ 'value-positive': budgetRemaining >= 0 }">{{ currencySymbol }}{{ budgetRemaining.toLocaleString(undefined, { minimumFractionDigits: 2, maximumFractionDigits: 2 }) }}</span>
              </div>
              <div class="summary-item">
                <span class="summary-label">Items</span>
                <span class="summary-value">{{ recommendedItems.length }}</span>
              </div>
            </div>
            <button
              class="btn-place-order"
              @click="placeOrder"
              :disabled="isSubmitting || submitSuccess"
            >
              {{ isSubmitting ? 'Submitting...' : submitSuccess ? 'Order Placed' : 'Place Order' }}
            </button>
          </div>
        </div>
      </div>

      <!-- Success / Error banners -->
      <div v-if="submitSuccess" class="success-banner">
        Order submitted successfully. Check the Orders tab to view delivery status and expected arrival date.
      </div>
      <div v-if="submitError" class="error">{{ submitError }}</div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const budget = ref(10000)
    const BUDGET_MIN = 1000
    const BUDGET_MAX = 100000
    const BUDGET_STEP = 500
    const inventoryItems = ref([])
    const demandForecasts = ref([])
    const loading = ref(true)
    const error = ref(null)
    const isSubmitting = ref(false)
    const submitError = ref(null)
    const submitSuccess = ref(false)

    const currencySymbol = computed(() => currentCurrency.value === 'JPY' ? '¥' : '$')

    const recommendedItems = computed(() => {
      // 1. Build demand trend lookup by SKU
      const forecastBySku = {}
      for (const f of demandForecasts.value) {
        forecastBySku[f.item_sku] = f.trend
      }
      // 2. Filter to items strictly below reorder_point
      const belowReorder = inventoryItems.value.filter(
        item => item.quantity_on_hand < item.reorder_point
      )
      // 3. Annotate with trend, qty_needed, line_cost
      const trendOrder = { increasing: 0, stable: 1, decreasing: 2 }
      const annotated = belowReorder.map(item => ({
        ...item,
        trend: forecastBySku[item.sku] || 'stable',
        qty_needed: item.reorder_point - item.quantity_on_hand,
        line_cost: (item.reorder_point - item.quantity_on_hand) * item.unit_cost
      }))
      // 4. Sort: increasing first, then stable, then decreasing
      annotated.sort((a, b) => trendOrder[a.trend] - trendOrder[b.trend])
      // 5. Greedy fill within budget (no partial quantities)
      let remaining = budget.value
      const selected = []
      for (const item of annotated) {
        if (item.line_cost <= remaining) {
          selected.push(item)
          remaining -= item.line_cost
        }
      }
      return selected
    })

    const totalCost = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.line_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - totalCost.value)

    const budgetUtilization = computed(() =>
      budget.value > 0 ? Math.min((totalCost.value / budget.value) * 100, 100) : 0
    )

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [inv, forecasts] = await Promise.all([
          api.getInventory(),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inv
        demandForecasts.value = forecasts
      } catch (err) {
        error.value = 'Failed to load restocking data. Please try again.'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (recommendedItems.value.length === 0 || isSubmitting.value) return
      isSubmitting.value = true
      submitError.value = null
      try {
        await api.submitRestockingOrder({
          budget: budget.value,
          items: recommendedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.qty_needed,
            unit_cost: item.unit_cost,
            total_cost: item.line_cost
          }))
        })
        submitSuccess.value = true
      } catch (err) {
        submitError.value = 'Failed to place order. Please try again.'
        console.error(err)
      } finally {
        isSubmitting.value = false
      }
    }

    watch(budget, () => {
      submitSuccess.value = false
      submitError.value = null
    })

    onMounted(loadData)

    return {
      budget,
      BUDGET_MIN,
      BUDGET_MAX,
      BUDGET_STEP,
      loading,
      error,
      inventoryItems,
      recommendedItems,
      totalCost,
      budgetRemaining,
      budgetUtilization,
      isSubmitting,
      submitError,
      submitSuccess,
      placeOrder,
      t,
      currencySymbol
    }
  }
}
</script>

<style scoped>
.budget-card .card-header {
  align-items: baseline;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
  margin: 1rem 0 0.5rem;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #94a3b8;
  margin-bottom: 1rem;
}

.utilization-section {
  margin-top: 0.5rem;
}

.utilization-bar-track {
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
  overflow: hidden;
}

.utilization-bar-fill {
  height: 100%;
  background: #2563eb;
  border-radius: 3px;
  transition: width 0.3s ease;
}

.utilization-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
  margin-top: 0.5rem;
}

.items-hint {
  font-size: 0.813rem;
  color: #94a3b8;
}

.restocking-table {
  table-layout: auto;
  width: 100%;
}

.col-num {
  text-align: right;
}

.sku-cell {
  font-family: 'Menlo', 'Consolas', monospace;
  font-size: 0.813rem;
  color: #475569;
}

.empty-state {
  padding: 3rem;
  text-align: center;
  color: #94a3b8;
}

.order-summary-bar {
  display: flex;
  align-items: center;
  justify-content: space-between;
  padding: 1rem 0.75rem;
  margin-top: 1rem;
  border-top: 1px solid #e2e8f0;
  background: #f8fafc;
  border-radius: 0 0 8px 8px;
}

.summary-totals {
  display: flex;
  gap: 2rem;
}

.summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.summary-label {
  font-size: 0.75rem;
  color: #64748b;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.04em;
}

.summary-value {
  font-size: 1.125rem;
  font-weight: 700;
  color: #0f172a;
}

.value-positive {
  color: #059669;
}

.btn-place-order {
  background: #2563eb;
  color: white;
  padding: 0.625rem 1.75rem;
  border-radius: 6px;
  font-weight: 600;
  font-size: 0.938rem;
  border: none;
  cursor: pointer;
  transition: background 0.2s;
}

.btn-place-order:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-place-order:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-place-order:disabled[data-success] {
  background: #059669;
  opacity: 1;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #a7f3d0;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-size: 0.938rem;
  font-weight: 500;
}
</style>
