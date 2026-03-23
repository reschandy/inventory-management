<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.budgetLabel') }}</h3>
        </div>
        <div style="padding: 1.5rem 0 0.5rem;">
          <input
            type="range"
            min="0"
            max="50000"
            step="500"
            v-model.number="budget"
            class="budget-slider"
          />
        </div>
        <div class="budget-stats">
          <div class="budget-stat">
            <div class="label">{{ t('restocking.budgetLabel') }}</div>
            <div class="value">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
          </div>
          <div class="budget-stat">
            <div class="label">{{ t('restocking.totalCost') }}</div>
            <div class="value">{{ currencySymbol }}{{ totalCost.toLocaleString() }}</div>
          </div>
          <div class="budget-stat">
            <div class="label">{{ t('restocking.remaining') }}</div>
            <div class="value" :style="{ color: budget - totalCost < 0 ? '#ef4444' : '#10b981' }">
              {{ currencySymbol }}{{ (budget - totalCost).toLocaleString() }}
            </div>
          </div>
        </div>
      </div>

      <!-- Success banner -->
      <div v-if="successMessage" class="success-banner">
        {{ successMessage }}
      </div>

      <!-- Error banner (post-submit errors) -->
      <div v-if="submitError" class="error">{{ submitError }}</div>

      <!-- Recommended Items card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">
            {{ t('restocking.selectedItems') }} ({{ selectedItems.length }})
          </h3>
        </div>
        <div v-if="selectedItems.length === 0" class="no-items">
          <p>{{ t('restocking.noItems') }}</p>
        </div>
        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.skuCol') }}</th>
                <th>{{ t('restocking.nameCol') }}</th>
                <th>{{ t('restocking.trendCol') }}</th>
                <th>{{ t('restocking.qtyCol') }}</th>
                <th>{{ t('restocking.unitCostCol') }}</th>
                <th>{{ t('restocking.totalCol') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="item in selectedItems" :key="item.item_sku">
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ item.item_name }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ item.trend }}</span>
                </td>
                <td>{{ item.forecasted_demand }}</td>
                <td>{{ currencySymbol }}{{ item.unit_cost.toLocaleString() }}</td>
                <td><strong>{{ currencySymbol }}{{ item.item_cost.toLocaleString() }}</strong></td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order button -->
      <div style="margin-top: 1.5rem;">
        <button
          class="place-order-btn"
          :disabled="selectedItems.length === 0 || submitting"
          @click="placeOrder"
        >
          {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
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
    const { t, currentCurrency } = useI18n()

    // Raw data
    const allForecasts = ref([])
    const inventoryMap = ref({})

    // UI state
    const budget = ref(10000)
    const loading = ref(true)
    const submitting = ref(false)
    const successMessage = ref(null)
    const error = ref(null)
    // Separate ref for post-submit errors so it doesn't replace the whole view
    const submitError = ref(null)

    // Currency symbol derived from locale
    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    // Filter and enrich forecasts: exclude decreasing trend, attach costs, sort by priority
    const eligibleForecasts = computed(() => {
      return allForecasts.value
        .filter(forecast => {
          // Only include non-decreasing trends that have matching inventory data
          return forecast.trend !== 'decreasing' && inventoryMap.value[forecast.item_sku]
        })
        .map(forecast => {
          const unit_cost = inventoryMap.value[forecast.item_sku].unit_cost
          const item_cost = forecast.forecasted_demand * unit_cost
          return { ...forecast, unit_cost, item_cost }
        })
        .sort((a, b) => {
          // Increasing trend first, then stable
          const tierOrder = { increasing: 0, stable: 1 }
          const tierA = tierOrder[a.trend] ?? 2
          const tierB = tierOrder[b.trend] ?? 2
          if (tierA !== tierB) return tierA - tierB
          // Within same tier, sort by item_cost ascending
          return a.item_cost - b.item_cost
        })
    })

    // Greedy selection: accumulate items while they fit within budget
    const selectedItems = computed(() => {
      const result = []
      let running = 0
      for (const item of eligibleForecasts.value) {
        if (running + item.item_cost <= budget.value) {
          result.push(item)
          running += item.item_cost
        }
      }
      return result
    })

    // Sum of all selected item costs
    const totalCost = computed(() => {
      return selectedItems.value.reduce((sum, item) => sum + item.item_cost, 0)
    })

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({})
        ])
        allForecasts.value = forecasts
        // Build a lookup map keyed by SKU for O(1) access in computed
        const map = {}
        for (const item of inventory) {
          map[item.sku] = item
        }
        inventoryMap.value = map
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    const placeOrder = async () => {
      submitting.value = true
      successMessage.value = null
      submitError.value = null
      try {
        const payload = {
          items: selectedItems.value.map(i => ({
            sku: i.item_sku,
            name: i.item_name,
            quantity: i.forecasted_demand,
            unit_price: i.unit_cost
          })),
          total_value: totalCost.value
        }
        await api.postRestockingOrder(payload)
        successMessage.value = t('restocking.success')
      } catch (err) {
        submitError.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => loadData())

    return {
      t,
      currencySymbol,
      allForecasts,
      inventoryMap,
      budget,
      loading,
      submitting,
      successMessage,
      error,
      submitError,
      eligibleForecasts,
      selectedItems,
      totalCost,
      loadData,
      placeOrder
    }
  }
}
</script>

<style scoped>
.budget-slider {
  width: 100%;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-stats {
  display: flex;
  gap: 2rem;
  flex-wrap: wrap;
  padding: 1rem 0 0.5rem;
}

.budget-stat {
  display: flex;
  flex-direction: column;
  gap: 0.25rem;
}

.budget-stat .label {
  font-size: 0.75rem;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  color: #64748b;
}

.budget-stat .value {
  font-size: 1.5rem;
  font-weight: 700;
  color: #0f172a;
}

.no-items {
  padding: 2.5rem;
  text-align: center;
  color: #64748b;
  font-size: 0.9375rem;
}

.place-order-btn {
  background-color: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  padding: 0.75rem 1.75rem;
  font-size: 0.9375rem;
  font-weight: 600;
  cursor: pointer;
  transition: background-color 0.15s ease, opacity 0.15s ease;
}

.place-order-btn:hover:not(:disabled) {
  background-color: #1d4ed8;
}

.place-order-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.success-banner {
  background-color: #d1fae5;
  color: #065f46;
  border: 1px solid #a7f3d0;
  border-radius: 8px;
  padding: 0.875rem 1.25rem;
  font-weight: 600;
  margin-bottom: 1.5rem;
}
</style>
