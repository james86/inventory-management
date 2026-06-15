<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <div v-if="orderPlaced" class="confirmation-banner">
        {{ t('restocking.orderPlaced') }}
      </div>

      <div class="card budget-card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budget') }}</h3>
          <span class="budget-amount">{{ formatCurrency(budget) }}</span>
        </div>
        <input
          type="range"
          class="budget-slider"
          min="0"
          max="250000"
          step="5000"
          v-model.number="budget"
        />
        <p class="budget-help">{{ t('restocking.budgetHelp') }}</p>
      </div>

      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.selectedItems') }}</div>
          <div class="stat-value">{{ selectedCount }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.selectedTotal') }}</div>
          <div class="stat-value">{{ formatCurrency(selectedTotal) }}</div>
        </div>
        <div :class="['stat-card', overBudget ? 'danger' : 'info']">
          <div class="stat-label">{{ t('restocking.remaining') }}</div>
          <div class="stat-value">{{ formatCurrency(remaining) }}</div>
        </div>
      </div>

      <div v-if="overBudget" class="error over-budget">{{ t('restocking.overBudget') }}</div>

      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendations') }}</h3>
        </div>
        <div v-if="recommendations.length === 0" class="empty-state">
          {{ t('restocking.noItems') }}
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.include') }}</th>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.itemName') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.orderQty') }}</th>
                <th>{{ t('restocking.table.unitCost') }}</th>
                <th>{{ t('restocking.table.lineTotal') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in recommendations" :key="item.id">
                <td>
                  <input
                    type="checkbox"
                    :checked="selectedIds.has(item.id)"
                    @change="toggleItem(item.id)"
                  />
                </td>
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">
                    {{ t(`trends.${item.trend}`) }}
                  </span>
                </td>
                <td>{{ item.current_demand }}</td>
                <td><strong>{{ item.forecasted_demand }}</strong></td>
                <td>{{ item.order_qty }}</td>
                <td>{{ formatCurrency(item.unit_cost) }}</td>
                <td><strong>{{ formatCurrency(item.line_total) }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <div class="actions">
        <span v-if="submitMessage" class="submit-message">{{ submitMessage }}</span>
        <button
          class="btn-secondary"
          :disabled="submitting || overBudget"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
        </button>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, reactive, onMounted, computed } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'
import { formatCurrency as formatCurrencyUtil } from '../utils/currency'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency } = useI18n()

    const loading = ref(true)
    const error = ref(null)
    const forecasts = ref([])
    const budget = ref(100000)
    const selectedIds = reactive(new Set())
    const submitting = ref(false)
    const orderPlaced = ref(false)
    const submitMessage = ref('')

    const formatCurrency = (value) => {
      return formatCurrencyUtil(value, currentCurrency.value)
    }

    // Items needing restock, with computed quantities, sorted for greedy fill
    const sortedCandidates = computed(() => {
      const candidates = forecasts.value
        .map(f => {
          const order_qty = Math.max(0, f.forecasted_demand - f.current_demand)
          return {
            ...f,
            order_qty,
            line_total: order_qty * f.unit_cost
          }
        })
        .filter(f => f.order_qty > 0)

      return candidates.sort((a, b) => {
        const aInc = a.trend === 'increasing' ? 0 : 1
        const bInc = b.trend === 'increasing' ? 0 : 1
        if (aInc !== bInc) return aInc - bInc
        // Within group: larger gap first
        const aGap = a.forecasted_demand - a.current_demand
        const bGap = b.forecasted_demand - b.current_demand
        return bGap - aGap
      })
    })

    const recommendations = computed(() => sortedCandidates.value)

    const selectedRows = computed(() =>
      recommendations.value.filter(item => selectedIds.has(item.id))
    )

    const selectedCount = computed(() => selectedRows.value.length)

    const selectedTotal = computed(() =>
      selectedRows.value.reduce((sum, item) => sum + item.line_total, 0)
    )

    const remaining = computed(() => budget.value - selectedTotal.value)

    const overBudget = computed(() => selectedTotal.value > budget.value)

    const toggleItem = (id) => {
      if (selectedIds.has(id)) {
        selectedIds.delete(id)
      } else {
        selectedIds.add(id)
      }
    }

    // Greedy default selection within budget
    const applyDefaultSelection = () => {
      selectedIds.clear()
      let runningTotal = 0
      for (const item of sortedCandidates.value) {
        if (runningTotal + item.line_total <= budget.value) {
          selectedIds.add(item.id)
          runningTotal += item.line_total
        }
      }
    }

    const loadForecasts = async () => {
      try {
        loading.value = true
        const data = await api.getDemandForecasts()
        forecasts.value = data
        applyDefaultSelection()
      } catch (err) {
        error.value = 'Failed to load demand forecasts: ' + err.message
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitMessage.value = ''
      orderPlaced.value = false

      if (recommendations.value.length === 0) {
        submitMessage.value = t('restocking.noItems')
        return
      }
      if (selectedRows.value.length === 0) {
        submitMessage.value = t('restocking.noneSelected')
        return
      }

      try {
        submitting.value = true
        const items = selectedRows.value.map(item => ({
          item_sku: item.item_sku,
          item_name: item.item_name,
          quantity: item.order_qty,
          unit_cost: item.unit_cost
        }))
        await api.submitRestockOrder({ budget: budget.value, items })
        orderPlaced.value = true
      } catch (err) {
        submitMessage.value = 'Failed to place order: ' + err.message
      } finally {
        submitting.value = false
      }
    }

    onMounted(loadForecasts)

    return {
      t,
      loading,
      error,
      budget,
      recommendations,
      selectedIds,
      selectedCount,
      selectedTotal,
      remaining,
      overBudget,
      submitting,
      orderPlaced,
      submitMessage,
      formatCurrency,
      toggleItem,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-card {
  margin-bottom: 1.5rem;
}

.budget-amount {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.budget-slider {
  -webkit-appearance: none;
  appearance: none;
  width: 100%;
  height: 8px;
  border-radius: 4px;
  background: #e2e8f0;
  outline: none;
  margin: 1rem 0 0.5rem;
  cursor: pointer;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #3b82f6;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  cursor: pointer;
}

.budget-slider::-moz-range-thumb {
  width: 24px;
  height: 24px;
  border-radius: 50%;
  background: #3b82f6;
  border: 3px solid white;
  box-shadow: 0 1px 3px rgba(0, 0, 0, 0.2);
  cursor: pointer;
}

.budget-help {
  font-size: 0.875rem;
  color: #64748b;
  margin: 0;
}

.confirmation-banner {
  background: #d1fae5;
  border: 1px solid #6ee7b7;
  color: #065f46;
  border-radius: 8px;
  padding: 1rem 1.5rem;
  margin-bottom: 1.5rem;
  font-weight: 500;
}

.over-budget {
  margin-bottom: 1.5rem;
}

.empty-state {
  padding: 2rem;
  text-align: center;
  color: #64748b;
}

.actions {
  display: flex;
  align-items: center;
  justify-content: flex-end;
  gap: 1rem;
  margin-top: 1.5rem;
}

.submit-message {
  font-size: 0.875rem;
  color: #ef4444;
  font-weight: 500;
}
</style>
