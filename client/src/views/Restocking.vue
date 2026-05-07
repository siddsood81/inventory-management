<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Greedy budget allocation across prioritized restock recommendations</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Budget</h3>
        </div>
        <div class="budget-body">
          <div class="budget-row">
            <label class="budget-label" for="budget-slider">Restock Budget</label>
            <span class="budget-value">{{ formatCurrency(budget) }}</span>
          </div>
          <input
            id="budget-slider"
            type="range"
            min="10000"
            max="500000"
            step="10000"
            v-model.number="budget"
            class="budget-slider"
          />
          <div class="budget-meta">
            <span class="budget-meta-item">
              <span class="budget-meta-label">Selected cost</span>
              <span class="budget-meta-value">{{ formatCurrency(selectedCost) }}</span>
            </span>
            <span class="budget-meta-item">
              <span class="budget-meta-label">Remaining</span>
              <span class="budget-meta-value remaining">{{ formatCurrency(remainingBudget) }}</span>
            </span>
            <span class="budget-meta-item">
              <span class="budget-meta-label">Items selected</span>
              <span class="budget-meta-value">{{ selectedItems.length }}</span>
            </span>
          </div>
        </div>
      </div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommendations ({{ recommendations.length }})</h3>
        </div>
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>SKU</th>
                <th>Name</th>
                <th>Category</th>
                <th>Warehouse</th>
                <th>Current Stock</th>
                <th>Reorder Point</th>
                <th>Restock Qty</th>
                <th>Unit Cost</th>
                <th>Est. Cost</th>
                <th>Priority</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'over-budget-row': isOverBudget(item) }"
              >
                <td><strong>{{ item.sku }}</strong></td>
                <td>
                  {{ item.name }}
                  <span v-if="item.priority === 'high'" class="badge info priority-badge">High Priority</span>
                </td>
                <td>{{ item.category }}</td>
                <td>{{ item.warehouse }}</td>
                <td>{{ item.current_stock }}</td>
                <td>{{ item.reorder_point }}</td>
                <td>{{ item.restock_quantity }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.estimated_cost) }}</strong></td>
                <td>
                  <span :class="['badge', item.priority]">{{ item.priority }}</span>
                </td>
                <td>
                  <span v-if="isOverBudget(item)" class="badge warning">Over budget</span>
                  <span v-else class="badge success">Selected</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div v-if="selectedItems.length > 0" class="card order-summary-card">
        <div class="card-header">
          <h3 class="card-title">Order Summary</h3>
        </div>
        <div class="summary-body">
          <div class="summary-stats">
            <div class="stat-card info">
              <div class="stat-label">Items</div>
              <div class="stat-value">{{ selectedItems.length }}</div>
            </div>
            <div class="stat-card">
              <div class="stat-label">Total Cost</div>
              <div class="stat-value total-cost-value">{{ formatCurrency(selectedCost) }}</div>
            </div>
            <div class="stat-card success">
              <div class="stat-label">Remaining Budget</div>
              <div class="stat-value">{{ formatCurrency(remainingBudget) }}</div>
            </div>
          </div>
          <div class="summary-action">
            <button
              class="place-order-btn"
              :disabled="selectedItems.length === 0 || submitting"
              @click="placeOrder"
            >
              {{ submitting ? 'Placing Order...' : 'Place Order' }}
            </button>
            <span v-if="orderError" class="order-error">{{ orderError }}</span>
          </div>
        </div>
      </div>

      <div v-if="submittedOrder" class="success-banner">
        <div class="success-banner-title">Order Placed Successfully</div>
        <div class="success-banner-details">
          <span>Order Number: <strong>{{ submittedOrder.order_number }}</strong></span>
          <span>Expected Delivery: <strong>{{ formatDate(submittedOrder.expected_delivery) }}</strong></span>
          <span>Total Value: <strong>{{ formatCurrency(submittedOrder.total_value) }}</strong></span>
        </div>
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
    const { currentCurrency, currentLocale } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const recommendations = ref([])
    const budget = ref(100000)
    const submitting = ref(false)
    const orderError = ref(null)
    const submittedOrder = ref(null)

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const formatCurrency = (value) => {
      return currencySymbol.value + Number(value).toLocaleString()
    }

    const formatDate = (dateString) => {
      const locale = currentLocale.value === 'ja' ? 'ja-JP' : 'en-US'
      return new Date(dateString).toLocaleDateString(locale, {
        year: 'numeric',
        month: 'short',
        day: 'numeric'
      })
    }

    const selectedItems = computed(() => {
      let runningTotal = 0
      const selected = []
      for (const item of recommendations.value) {
        if (runningTotal + item.estimated_cost <= budget.value) {
          runningTotal += item.estimated_cost
          selected.push(item)
        }
      }
      return selected
    })

    const selectedSkus = computed(() => new Set(selectedItems.value.map(i => i.sku)))

    const isOverBudget = (item) => !selectedSkus.value.has(item.sku)

    const selectedCost = computed(() =>
      selectedItems.value.reduce((sum, item) => sum + item.estimated_cost, 0)
    )

    const remainingBudget = computed(() => budget.value - selectedCost.value)

    const loadRecommendations = async () => {
      try {
        loading.value = true
        error.value = null
        recommendations.value = await api.getRestockingRecommendations()
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      if (selectedItems.value.length === 0 || submitting.value) return
      submitting.value = true
      orderError.value = null
      submittedOrder.value = null
      try {
        const items = selectedItems.value.map(i => ({
          sku: i.sku,
          name: i.name,
          quantity: i.restock_quantity,
          unit_cost: i.unit_cost
        }))
        const warehouse = selectedItems.value[0]?.warehouse || 'San Francisco'
        const order = await api.submitRestockingOrder(items, warehouse)
        submittedOrder.value = order
      } catch (err) {
        orderError.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadRecommendations)

    return {
      loading,
      error,
      recommendations,
      budget,
      submitting,
      orderError,
      submittedOrder,
      selectedItems,
      selectedCost,
      remainingBudget,
      isOverBudget,
      formatCurrency,
      formatDate,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-body {
  padding: 0.5rem 0;
}

.budget-row {
  display: flex;
  align-items: center;
  justify-content: space-between;
  margin-bottom: 0.75rem;
}

.budget-label {
  font-size: 0.938rem;
  font-weight: 600;
  color: #0f172a;
}

.budget-value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  height: 6px;
  margin-bottom: 1.25rem;
  cursor: pointer;
}

.budget-meta {
  display: flex;
  gap: 2rem;
  flex-wrap: wrap;
}

.budget-meta-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-meta-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.budget-meta-value {
  font-size: 1rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-meta-value.remaining {
  color: #059669;
}

.over-budget-row {
  opacity: 0.45;
}

.priority-badge {
  margin-left: 0.5rem;
  font-size: 0.688rem;
  padding: 0.2rem 0.5rem;
}

.order-summary-card {
  border-color: #bfdbfe;
}

.summary-body {
  display: flex;
  flex-direction: column;
  gap: 1.25rem;
}

.summary-stats {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(180px, 1fr));
  gap: 1rem;
}

.total-cost-value {
  font-size: 1.75rem;
  color: #2563eb;
}

.summary-action {
  display: flex;
  align-items: center;
  gap: 1.25rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  border: none;
  font-size: 0.938rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.order-error {
  color: #991b1b;
  font-size: 0.875rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  border-radius: 10px;
  padding: 1.25rem 1.5rem;
  margin-bottom: 1.25rem;
}

.success-banner-title {
  font-size: 1rem;
  font-weight: 700;
  color: #065f46;
  margin-bottom: 0.625rem;
}

.success-banner-details {
  display: flex;
  gap: 2rem;
  flex-wrap: wrap;
  font-size: 0.875rem;
  color: #065f46;
}
</style>
