<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking Planner</h2>
      <p>Recommends items to restock based on your available budget and current stock levels.</p>
    </div>

    <div v-if="loading" class="loading">Loading inventory data...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget card -->
      <div class="card">
        <div class="card-header">
          <span class="card-title">Budget</span>
          <span class="budget-display">{{ formatCurrency(budget) }}</span>
        </div>
        <div class="budget-controls">
          <div class="slider-row">
            <span class="slider-label">$0</span>
            <input
              type="range"
              v-model.number="budget"
              min="0"
              max="500000"
              step="5000"
              class="budget-slider"
            />
            <span class="slider-label">$500,000</span>
          </div>
          <div class="budget-remaining">
            Remaining: <strong>{{ formatCurrency(budgetRemaining) }}</strong>
          </div>
        </div>
      </div>

      <!-- Stat cards -->
      <div class="stats-grid">
        <div class="stat-card warning">
          <div class="stat-label">Low Stock Items</div>
          <div class="stat-value">{{ lowStockCount }}</div>
        </div>
        <div class="stat-card info">
          <div class="stat-label">Items to Restock</div>
          <div class="stat-value">{{ recommendedItems.length }}</div>
        </div>
        <div class="stat-card danger">
          <div class="stat-label">Budget Used</div>
          <div class="stat-value budget-used-value">{{ formatCurrency(budgetUsed) }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">Remaining Budget</div>
          <div class="stat-value budget-remaining-value">{{ formatCurrency(budgetRemaining) }}</div>
        </div>
      </div>

      <!-- Recommendations card -->
      <div class="card">
        <div class="card-header">
          <span class="card-title">Recommended Items to Restock</span>
          <span v-if="recommendedItems.length > 0" class="item-count">
            {{ recommendedItems.length }} item{{ recommendedItems.length !== 1 ? 's' : '' }}
          </span>
        </div>

        <div v-if="recommendedItems.length > 0">
          <div class="table-container">
            <table>
              <thead>
                <tr>
                  <th>Item Name</th>
                  <th>SKU</th>
                  <th>On Hand</th>
                  <th>Reorder Point</th>
                  <th>Restock Qty</th>
                  <th>Unit Cost</th>
                  <th>Total Cost</th>
                </tr>
              </thead>
              <tbody>
                <tr v-for="item in recommendedItems" :key="item.sku">
                  <td><strong>{{ item.name }}</strong></td>
                  <td>
                    <span class="badge info">{{ item.sku }}</span>
                  </td>
                  <td>
                    <span class="badge danger">{{ item.quantity_on_hand }}</span>
                  </td>
                  <td>{{ item.reorder_point }}</td>
                  <td><strong>{{ item.restock_qty }}</strong></td>
                  <td>{{ formatCurrency(item.unit_cost) }}</td>
                  <td><strong>{{ formatCurrency(item.restock_cost) }}</strong></td>
                </tr>
              </tbody>
            </table>
          </div>
        </div>

        <div v-else class="empty-state">
          No items can be restocked within the current budget.
        </div>

        <!-- Place Order section -->
        <div class="order-actions">
          <button
            class="place-order-btn"
            :disabled="recommendedItems.length === 0 || submitting"
            @click="placeOrder"
          >
            {{ submitting ? 'Placing Order...' : 'Place Order' }}
          </button>

          <div v-if="successMessage" class="success-banner">
            {{ successMessage }}
          </div>
          <div v-if="errorMessage" class="error">
            {{ errorMessage }}
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'

export default {
  name: 'Restocking',
  setup() {
    const loading = ref(false)
    const error = ref(null)
    const inventoryItems = ref([])
    const forecasts = ref([])
    const budget = ref(100000)
    const submitting = ref(false)
    const successMessage = ref(null)
    const errorMessage = ref(null)

    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const formatCurrency = (value) =>
      value.toLocaleString('en-US', { style: 'currency', currency: 'USD' })

    const lowStockCount = computed(() =>
      inventoryItems.value.filter(item => item.quantity_on_hand <= item.reorder_point).length
    )

    const recommendedItems = computed(() => {
      let remaining = budget.value
      const result = []

      const lowStock = inventoryItems.value
        .filter(item => item.quantity_on_hand <= item.reorder_point)
        .map(item => {
          const forecast = forecasts.value.find(f => f.item_sku === item.sku)
          const qty = forecast ? forecast.forecasted_demand : item.reorder_point
          const cost = qty * item.unit_cost
          return { ...item, restock_qty: qty, restock_cost: cost }
        })
        .sort((a, b) =>
          (b.reorder_point - b.quantity_on_hand) - (a.reorder_point - a.quantity_on_hand)
        )

      for (const item of lowStock) {
        if (item.restock_cost <= remaining) {
          result.push(item)
          remaining -= item.restock_cost
        }
      }
      return result
    })

    const budgetUsed = computed(() =>
      recommendedItems.value.reduce((sum, item) => sum + item.restock_cost, 0)
    )

    const budgetRemaining = computed(() => budget.value - budgetUsed.value)

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const [inv, fc] = await Promise.all([
          api.getInventory({ warehouse: filters.warehouse, category: filters.category }),
          api.getDemandForecasts()
        ])
        inventoryItems.value = inv
        forecasts.value = fc
      } catch (err) {
        error.value = 'Failed to load data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      successMessage.value = null
      errorMessage.value = null
      try {
        const orderData = {
          customer: 'Internal Restock',
          items: recommendedItems.value.map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.restock_qty,
            unit_price: item.unit_cost
          })),
          warehouse: null,
          category: null
        }
        const order = await api.createRestockOrder(orderData)
        successMessage.value = `Order ${order.order_number} placed successfully. Expected delivery in 14 days.`
      } catch (err) {
        errorMessage.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    watch([selectedLocation, selectedCategory], loadData)

    onMounted(loadData)

    return {
      loading,
      error,
      inventoryItems,
      forecasts,
      budget,
      submitting,
      successMessage,
      errorMessage,
      formatCurrency,
      lowStockCount,
      recommendedItems,
      budgetUsed,
      budgetRemaining,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

.budget-controls {
  padding: 0.5rem 0 0.25rem;
}

.budget-display {
  font-size: 1.5rem;
  font-weight: 700;
  color: #2563eb;
  letter-spacing: -0.025em;
}

.slider-row {
  display: flex;
  align-items: center;
  gap: 1rem;
  margin-bottom: 0.75rem;
}

.slider-label {
  font-size: 0.813rem;
  color: #64748b;
  white-space: nowrap;
  min-width: 4rem;
}

.slider-label:last-child {
  text-align: right;
}

.budget-slider {
  flex: 1;
  -webkit-appearance: none;
  appearance: none;
  height: 6px;
  border-radius: 3px;
  background: #e2e8f0;
  outline: none;
  cursor: pointer;
  transition: background 0.2s;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
  transition: box-shadow 0.2s;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 6px rgba(37, 99, 235, 0.15);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: none;
  box-shadow: 0 1px 4px rgba(37, 99, 235, 0.4);
}

.budget-remaining {
  font-size: 0.938rem;
  color: #64748b;
}

.budget-remaining strong {
  color: #059669;
}

.item-count {
  font-size: 0.875rem;
  color: #64748b;
  font-weight: 500;
}

.budget-used-value {
  font-size: 1.5rem;
}

.budget-remaining-value {
  font-size: 1.5rem;
}

.empty-state {
  padding: 3rem 1rem;
  text-align: center;
  color: #94a3b8;
  font-size: 0.938rem;
}

.order-actions {
  padding-top: 1.25rem;
  border-top: 1px solid #e2e8f0;
  margin-top: 1.25rem;
}

.place-order-btn {
  background: #2563eb;
  color: white;
  padding: 0.75rem 2rem;
  border-radius: 8px;
  font-weight: 600;
  font-size: 0.938rem;
  border: none;
  cursor: pointer;
  transition: background 0.2s, opacity 0.2s;
}

.place-order-btn:hover:not(:disabled) {
  background: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.45;
  cursor: not-allowed;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem;
  border-radius: 8px;
  margin-top: 1rem;
  font-size: 0.938rem;
}
</style>
