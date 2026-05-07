<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Set your budget to get AI-powered restocking recommendations based on demand forecasts and current stock levels.</p>
    </div>

    <div class="card budget-card">
      <div class="budget-label">Available Budget</div>
      <div class="budget-amount">{{ formatCurrency(budget) }}</div>
      <input
        type="range"
        class="budget-slider"
        min="0"
        max="500000"
        step="5000"
        v-model.number="budget"
      />
      <div class="budget-hint">Drag to set your restocking budget</div>
    </div>

    <div v-if="recommendations.length > 0 || totalCost > 0" class="budget-summary-bar">
      <div class="budget-summary-item">
        <span class="summary-label">Budget</span>
        <span class="summary-value">{{ formatCurrency(budget) }}</span>
      </div>
      <div class="budget-summary-item">
        <span class="summary-label">Allocated</span>
        <span class="summary-value" :style="{ color: totalCost > budget * 0.8 ? '#dc2626' : '#059669' }">
          {{ formatCurrency(totalCost) }}
        </span>
      </div>
      <div class="budget-summary-item">
        <span class="summary-label">Remaining</span>
        <span class="summary-value" :style="{ color: remainingBudget >= 0 ? '#059669' : '#dc2626' }">
          {{ formatCurrency(remainingBudget) }}
        </span>
      </div>
    </div>

    <div v-if="submittedOrder" class="success-banner">
      <div class="success-title">
        Order {{ submittedOrder.order_number }} placed successfully — expected delivery in 14 days.
      </div>
      <div class="success-sub">Navigate to the Orders tab to see your submitted order.</div>
    </div>

    <div v-if="isLoading" class="loading">Loading recommendations...</div>

    <div v-else-if="error" class="error">{{ error }}</div>

    <div v-else-if="recommendations.length > 0" class="card">
      <div class="card-header">
        <h3 class="card-title">Restocking Recommendations ({{ recommendations.length }} items)</h3>
        <button
          class="btn-primary"
          :disabled="isSubmitting"
          @click="placeOrder"
        >
          {{ isSubmitting ? 'Placing...' : 'Place Order' }}
        </button>
      </div>
      <div class="table-container">
        <table class="restock-table">
          <thead>
            <tr>
              <th class="col-item">Item</th>
              <th class="col-warehouse">Warehouse</th>
              <th class="col-stock">Stock</th>
              <th class="col-demand">Forecasted Demand</th>
              <th class="col-qty">Restock Qty</th>
              <th class="col-unit">Unit Cost</th>
              <th class="col-total">Total Cost</th>
              <th class="col-trend">Trend</th>
            </tr>
          </thead>
          <tbody>
            <tr v-for="item in recommendations" :key="item.sku">
              <td class="col-item">
                <span class="item-name">{{ item.name }}</span>
                <span class="item-sku">{{ item.sku }}</span>
              </td>
              <td class="col-warehouse">{{ item.warehouse }}</td>
              <td class="col-stock">
                {{ item.current_stock }} / {{ item.reorder_point }}
                <span v-if="item.is_below_reorder" class="badge danger low-stock-badge">Low Stock</span>
              </td>
              <td class="col-demand">{{ item.forecasted_demand }}</td>
              <td class="col-qty">{{ item.restock_quantity }}</td>
              <td class="col-unit">{{ formatCurrency(item.unit_cost) }}</td>
              <td class="col-total"><strong>{{ formatCurrency(item.restock_cost) }}</strong></td>
              <td class="col-trend">
                <span :class="['badge', getTrendClass(item.trend)]">{{ item.trend }}</span>
              </td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>

    <div v-else-if="!isLoading && !submittedOrder" class="empty-state">
      <div class="empty-title">No recommendations for this budget.</div>
      <div class="empty-sub">Try increasing your budget or adjusting filters.</div>
    </div>
  </div>
</template>

<script>
import { ref, watch, onMounted } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'

export default {
  name: 'Restocking',
  setup() {
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    const budget = ref(50000)
    const recommendations = ref([])
    const totalCost = ref(0)
    const remainingBudget = ref(0)
    const isLoading = ref(false)
    const isSubmitting = ref(false)
    const submittedOrder = ref(null)
    const error = ref(null)

    const formatCurrency = (val) => {
      return '$' + val.toLocaleString('en-US', { minimumFractionDigits: 0, maximumFractionDigits: 0 })
    }

    const getTrendClass = (trend) => {
      if (!trend) return 'info'
      const lower = trend.toLowerCase()
      if (lower === 'increasing') return 'increasing'
      if (lower === 'decreasing') return 'decreasing'
      return 'stable'
    }

    const loadRecommendations = async () => {
      isLoading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const response = await api.getRestockRecommendations(budget.value, {
          warehouse: filters.warehouse,
          category: filters.category
        })
        recommendations.value = response.recommendations || []
        totalCost.value = response.total_cost || 0
        remainingBudget.value = response.remaining_budget || 0
      } catch (err) {
        error.value = 'Failed to load recommendations'
        console.error(err)
      } finally {
        isLoading.value = false
      }
    }

    const placeOrder = async () => {
      isSubmitting.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const items = recommendations.value.map(r => ({
          sku: r.sku,
          name: r.name,
          quantity: r.restock_quantity,
          unit_price: r.unit_cost
        }))
        const response = await api.createOrder({
          customer: 'Restocking System',
          items,
          status: 'Submitted',
          warehouse: filters.warehouse,
          category: filters.category
        })
        submittedOrder.value = response
        recommendations.value = []
      } catch (err) {
        error.value = 'Failed to place order'
        console.error(err)
      } finally {
        isSubmitting.value = false
      }
    }

    let debounceTimer = null
    watch(budget, () => {
      clearTimeout(debounceTimer)
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 300)
    })

    watch([selectedLocation, selectedCategory], () => {
      loadRecommendations()
    })

    onMounted(() => loadRecommendations())

    return {
      budget,
      recommendations,
      totalCost,
      remainingBudget,
      isLoading,
      isSubmitting,
      submittedOrder,
      error,
      formatCurrency,
      getTrendClass,
      loadRecommendations,
      placeOrder
    }
  }
}
</script>

<style scoped>
.restocking {
  padding: 0;
}

.budget-card {
  margin-bottom: 1.25rem;
}

.budget-label {
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.5rem;
}

.budget-amount {
  font-size: 2rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
}

.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
  margin-bottom: 0.5rem;
}

.budget-hint {
  font-size: 0.813rem;
  color: #64748b;
}

.budget-summary-bar {
  display: flex;
  gap: 2rem;
  background: white;
  border: 1px solid #e2e8f0;
  border-radius: 10px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
}

.budget-summary-item {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.summary-label {
  font-size: 0.75rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
}

.summary-value {
  font-size: 1.25rem;
  font-weight: 700;
  color: #0f172a;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 10px;
  padding: 1rem 1.25rem;
  margin-bottom: 1.25rem;
}

.success-title {
  font-weight: 600;
  font-size: 0.938rem;
  margin-bottom: 0.25rem;
}

.success-sub {
  font-size: 0.875rem;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.5rem 1.25rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.2s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.restock-table {
  table-layout: fixed;
  width: 100%;
}

.col-item { width: 200px; }
.col-warehouse { width: 120px; }
.col-stock { width: 150px; }
.col-demand { width: 130px; }
.col-qty { width: 100px; }
.col-unit { width: 110px; }
.col-total { width: 110px; }
.col-trend { width: 110px; }

.item-name {
  display: block;
  font-weight: 500;
  color: #0f172a;
  font-size: 0.875rem;
}

.item-sku {
  display: block;
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 0.125rem;
}

.low-stock-badge {
  display: inline-block;
  margin-left: 0.375rem;
  font-size: 0.688rem;
  padding: 0.125rem 0.5rem;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
}

.empty-title {
  font-size: 1rem;
  font-weight: 600;
  color: #334155;
  margin-bottom: 0.5rem;
}

.empty-sub {
  font-size: 0.875rem;
  color: #64748b;
}
</style>
