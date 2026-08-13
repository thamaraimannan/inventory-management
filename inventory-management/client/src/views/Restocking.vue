<script>
import { ref, computed } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t } = useI18n()

    const budget = ref(100000)
    const recommendations = ref([])
    const loading = ref(false)
    const error = ref(null)
    const isPlacingOrder = ref(false)
    const successOrder = ref(null)

    const totalCost = computed(() => {
      return recommendations.value.reduce((sum, item) => sum + item.line_total, 0)
    })

    const remainingBudget = computed(() => {
      return budget.value - totalCost.value
    })

    const isOverBudget = computed(() => remainingBudget.value < 0)

    const formatCurrency = (value) => {
      return value.toLocaleString('en-US', { style: 'currency', currency: 'USD', maximumFractionDigits: 2 })
    }

    const formatBudgetDisplay = (value) => {
      return '$' + value.toLocaleString('en-US')
    }

    const getRecommendations = async () => {
      loading.value = true
      error.value = null
      successOrder.value = null
      try {
        recommendations.value = await api.getRestockingRecommendations(budget.value)
      } catch (err) {
        error.value = t('common.error') + ': Failed to load recommendations'
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      isPlacingOrder.value = true
      error.value = null
      try {
        const order = await api.placeRestockingOrder({ budget: budget.value, items: recommendations.value })
        successOrder.value = order
        recommendations.value = []
      } catch (err) {
        error.value = t('common.error') + ': Failed to place order'
        console.error(err)
      } finally {
        isPlacingOrder.value = false
      }
    }

    return {
      t,
      budget,
      recommendations,
      loading,
      error,
      isPlacingOrder,
      successOrder,
      totalCost,
      remainingBudget,
      isOverBudget,
      formatCurrency,
      formatBudgetDisplay,
      getRecommendations,
      placeOrder
    }
  }
}
</script>

<template>
  <div class="view-container">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Success Banner -->
    <div v-if="successOrder" class="success-banner">
      <span>{{ t('restocking.orderSuccess', { orderNumber: successOrder.order_number }) }}</span>
      <router-link to="/orders" class="btn-secondary" style="text-decoration: none; display: inline-block;">
        {{ t('restocking.viewInOrders') }}
      </router-link>
    </div>

    <!-- Error State -->
    <div v-if="error" class="error">{{ error }}</div>

    <!-- Budget Card -->
    <div class="card">
      <div class="card-header">
        <span class="card-title">{{ t('restocking.budgetLabel') }}</span>
      </div>

      <div class="budget-display">{{ formatBudgetDisplay(budget) }}</div>

      <div class="slider-wrapper">
        <input
          type="range"
          class="slider-input"
          min="0"
          max="1000000"
          step="1000"
          v-model.number="budget"
        />
        <div class="slider-labels">
          <span>$0</span>
          <span>$1,000,000</span>
        </div>
      </div>

      <div class="action-row">
        <button class="btn-primary" :disabled="loading" @click="getRecommendations">
          <span v-if="loading">{{ t('common.loading') }}</span>
          <span v-else>{{ t('restocking.getRecommendations') }}</span>
        </button>
      </div>
    </div>

    <!-- Recommendations Card -->
    <div v-if="!loading" class="card">
      <div class="card-header recommendations-header">
        <span class="card-title">{{ t('restocking.recommendedItems') }}</span>
        <div v-if="recommendations.length > 0" class="budget-summary">
          <span>
            {{ t('restocking.totalCost') }}:
            <strong :class="{ 'over-budget': isOverBudget }">{{ formatCurrency(totalCost) }}</strong>
            / {{ formatBudgetDisplay(budget) }}
          </span>
          <span>
            {{ t('restocking.remainingBudget') }}:
            <strong :class="{ 'over-budget': isOverBudget }">{{ formatCurrency(remainingBudget) }}</strong>
          </span>
        </div>
      </div>

      <!-- Empty State: Not yet fetched -->
      <div v-if="recommendations.length === 0 && !successOrder" class="empty-state">
        {{ t('restocking.adjustBudget') }}
      </div>

      <!-- Recommendations Table -->
      <div v-if="recommendations.length > 0">
        <div class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.onHand') }}</th>
                <th>{{ t('restocking.table.reorderPoint') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.restockQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
                <th>{{ t('restocking.table.priority') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.item_sku">
                <td>{{ item.item_sku }}</td>
                <td>{{ item.item_name }}</td>
                <td>{{ item.quantity_on_hand.toLocaleString() }}</td>
                <td>{{ item.reorder_point.toLocaleString() }}</td>
                <td>{{ item.forecasted_demand.toLocaleString() }}</td>
                <td><span :class="['badge', item.trend]">{{ item.trend }}</span></td>
                <td>{{ item.restock_quantity.toLocaleString() }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td>{{ formatCurrency(item.line_total) }}</td>
                <td><span :class="['badge', item.priority]">{{ item.priority }}</span></td>
              </tr>
            </tbody>
          </table>
        </div>

        <div class="action-row">
          <button
            class="btn-primary"
            :disabled="isPlacingOrder"
            @click="placeOrder"
          >
            <span v-if="isPlacingOrder">{{ t('restocking.placingOrder') }}</span>
            <span v-else>{{ t('restocking.placeOrder') }}</span>
          </button>
        </div>
      </div>
    </div>

    <!-- Loading State inside card area -->
    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
  </div>
</template>

<style scoped>
.view-container {
  padding: 2rem;
}

.budget-display {
  font-size: 2rem;
  font-weight: 700;
  color: #2563eb;
  margin-bottom: 0.5rem;
}

.slider-wrapper {
  margin: 0.75rem 0;
}

.slider-input {
  width: 100%;
  accent-color: #2563eb;
}

.slider-labels {
  display: flex;
  justify-content: space-between;
  font-size: 0.75rem;
  color: #64748b;
  margin-top: 0.25rem;
}

.budget-summary {
  display: flex;
  gap: 2rem;
  margin-top: 1rem;
  font-size: 0.875rem;
  color: #64748b;
}

.budget-summary strong {
  color: #0f172a;
}

.over-budget {
  color: #dc2626 !important;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.6;
  cursor: not-allowed;
}

.btn-secondary {
  background: white;
  color: #374151;
  border: 1px solid #d1d5db;
  padding: 0.625rem 1.5rem;
  border-radius: 6px;
  font-size: 0.875rem;
  font-weight: 500;
  cursor: pointer;
  margin-right: 0.75rem;
}

.btn-secondary:hover {
  background: #f9fafb;
}

.action-row {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  margin-top: 1rem;
  gap: 0.75rem;
}

.empty-state {
  text-align: center;
  padding: 3rem;
  color: #64748b;
  font-size: 0.938rem;
}

.success-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  padding: 1rem 1.5rem;
  border-radius: 8px;
  margin-bottom: 1rem;
  font-weight: 500;
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.recommendations-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
}
</style>
