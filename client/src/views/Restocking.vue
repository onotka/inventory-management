<template>
  <div class="restocking">
    <div class="page-header">
      <h2>{{ t('restocking.title') }}</h2>
      <p>{{ t('restocking.description') }}</p>
    </div>

    <!-- Budget slider card -->
    <div class="card budget-card">
      <label class="budget-label">{{ t('restocking.budgetLabel') }}</label>
      <div class="budget-display">{{ currencySymbol }}{{ budget.toLocaleString() }}</div>
      <input
        type="range"
        min="1000"
        max="100000"
        step="500"
        v-model.number="budget"
        class="budget-slider"
      />
      <div class="budget-bounds">
        <span>{{ currencySymbol }}1,000</span>
        <span>{{ currencySymbol }}100,000</span>
      </div>
      <p class="budget-hint">{{ t('restocking.budgetHint') }}</p>
    </div>

    <!-- Loading state (first load only) -->
    <div v-if="loading && !recommendations.items.length" class="loading">
      {{ t('common.loading') }}
    </div>

    <!-- Stats and table (always visible once data loaded) -->
    <template v-if="!loading || recommendations.items.length">
      <!-- Stat cards -->
      <div class="stats-grid">
        <div class="stat-card info">
          <div class="stat-label">{{ t('restocking.recommendedItems') }}</div>
          <div class="stat-value">{{ pickedCount }}</div>
        </div>
        <div class="stat-card success">
          <div class="stat-label">{{ t('restocking.totalCost') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ recommendations.total_cost.toLocaleString() }}</div>
        </div>
        <div class="stat-card warning">
          <div class="stat-label">{{ t('restocking.budgetRemaining') }}</div>
          <div class="stat-value">{{ currencySymbol }}{{ recommendations.budget_remaining.toLocaleString() }}</div>
        </div>
      </div>

      <!-- Success banner -->
      <div v-if="successInfo" class="success-banner">
        <span>{{ t('restocking.successCreated', { orderNumber: successInfo.orderNumber, expectedDelivery: successInfo.expectedDelivery }) }}</span>
        <button class="close-btn" @click="successInfo = null">&times;</button>
      </div>

      <!-- Error banner -->
      <div v-if="error" class="error">{{ error }}</div>

      <!-- Empty state -->
      <div v-if="!loading && pickedCount === 0 && recommendations.items.length === 0" class="empty-banner">
        {{ t('restocking.noRecommendations') }}
      </div>

      <!-- Recommendations table card -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('restocking.recommendedItems') }} ({{ recommendations.items.length }})</h3>
          <button
            class="btn-primary"
            :disabled="!pickedCount || submitting"
            @click="placeOrder"
          >
            {{ submitting ? t('restocking.placing') : t('restocking.placeOrder') }}
          </button>
        </div>

        <div v-if="loading" class="table-loading">{{ t('common.loading') }}</div>

        <div v-else class="table-container">
          <table>
            <thead>
              <tr>
                <th>{{ t('restocking.table.sku') }}</th>
                <th>{{ t('restocking.table.name') }}</th>
                <th>{{ t('restocking.table.trend') }}</th>
                <th>{{ t('restocking.table.currentDemand') }}</th>
                <th>{{ t('restocking.table.forecastedDemand') }}</th>
                <th>{{ t('restocking.table.quantity') }}</th>
                <th>{{ t('restocking.table.unitPrice') }}</th>
                <th>{{ t('restocking.table.lineCost') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations.items"
                :key="item.item_sku"
                :class="{ muted: item.recommended_quantity === 0 }"
              >
                <td><strong>{{ item.item_sku }}</strong></td>
                <td>{{ translateProductName(item.item_name) }}</td>
                <td>
                  <span :class="['badge', item.trend]">{{ t('trends.' + item.trend) }}</span>
                </td>
                <td>{{ item.current_demand }}</td>
                <td>{{ item.forecasted_demand }}</td>
                <td><strong>{{ item.recommended_quantity }}</strong></td>
                <td>{{ currencySymbol }}{{ item.unit_price.toFixed(2) }}</td>
                <td>{{ currencySymbol }}{{ item.line_cost.toLocaleString() }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </template>
  </div>
</template>

<script>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import { api } from '../api'
import { useI18n } from '../composables/useI18n'

export default {
  name: 'Restocking',
  setup() {
    const { t, currentCurrency, translateProductName } = useI18n()

    const currencySymbol = computed(() => {
      return currentCurrency.value === 'JPY' ? '¥' : '$'
    })

    const budget = ref(20000)
    const loading = ref(false)
    const error = ref(null)
    const submitting = ref(false)
    const successInfo = ref(null)

    const recommendations = ref({
      items: [],
      total_cost: 0,
      budget: 0,
      budget_remaining: 0
    })

    let debounceTimer = null

    const pickedCount = computed(() => {
      return recommendations.value.items.filter(item => item.recommended_quantity > 0).length
    })

    const loadRecommendations = async () => {
      loading.value = true
      error.value = null
      try {
        const result = await api.getRestockingRecommendations(budget.value)
        recommendations.value = result
      } catch (err) {
        error.value = 'Failed to load restocking recommendations: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    watch(budget, () => {
      if (debounceTimer) clearTimeout(debounceTimer)
      loading.value = true
      debounceTimer = setTimeout(() => {
        loadRecommendations()
      }, 250)
    })

    const placeOrder = async () => {
      submitting.value = true
      error.value = null
      try {
        const items = recommendations.value.items
          .filter(item => item.recommended_quantity > 0)
          .map(item => ({
            item_sku: item.item_sku,
            item_name: item.item_name,
            quantity: item.recommended_quantity,
            unit_price: item.unit_price
          }))

        const payload = {
          items,
          budget: budget.value
        }

        const result = await api.submitRestockingOrder(payload)
        successInfo.value = {
          orderNumber: result.order_number,
          expectedDelivery: result.expected_delivery
        }

        setTimeout(() => {
          successInfo.value = null
        }, 6000)

        await loadRecommendations()
      } catch (err) {
        error.value = err.message
        console.error(err)
      } finally {
        submitting.value = false
      }
    }

    onMounted(() => {
      loadRecommendations()
    })

    onBeforeUnmount(() => {
      if (debounceTimer) clearTimeout(debounceTimer)
    })

    return {
      t,
      currencySymbol,
      budget,
      loading,
      error,
      submitting,
      successInfo,
      recommendations,
      pickedCount,
      placeOrder,
      translateProductName
    }
  }
}
</script>

<style scoped>
.restocking {
  padding-bottom: 2rem;
}

/* Budget card */
.budget-card {
  margin-bottom: 1.5rem;
  padding: 1.75rem;
}

.budget-label {
  display: block;
  font-size: 0.875rem;
  font-weight: 600;
  color: #64748b;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  margin-bottom: 0.75rem;
}

.budget-display {
  font-size: 2.25rem;
  font-weight: 700;
  color: #0f172a;
  letter-spacing: -0.025em;
  margin-bottom: 1rem;
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
  margin-bottom: 0.5rem;
}

.budget-slider::-webkit-slider-thumb {
  -webkit-appearance: none;
  appearance: none;
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 0 0 2px #2563eb, 0 2px 4px rgba(37, 99, 235, 0.3);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-webkit-slider-thumb:hover {
  box-shadow: 0 0 0 3px #2563eb, 0 2px 8px rgba(37, 99, 235, 0.4);
}

.budget-slider::-moz-range-thumb {
  width: 20px;
  height: 20px;
  border-radius: 50%;
  background: #2563eb;
  cursor: pointer;
  border: 2px solid #ffffff;
  box-shadow: 0 0 0 2px #2563eb, 0 2px 4px rgba(37, 99, 235, 0.3);
  transition: box-shadow 0.15s ease;
}

.budget-slider::-moz-range-track {
  height: 6px;
  background: #e2e8f0;
  border-radius: 3px;
}

.budget-bounds {
  display: flex;
  justify-content: space-between;
  font-size: 0.813rem;
  color: #64748b;
  margin-bottom: 0.75rem;
}

.budget-hint {
  font-size: 0.875rem;
  color: #64748b;
  margin: 0;
}

/* Place order button */
.btn-primary {
  background: #2563eb;
  color: #ffffff;
  border: none;
  border-radius: 8px;
  padding: 0.625rem 1.25rem;
  font-size: 0.875rem;
  font-weight: 600;
  cursor: pointer;
  transition: background 0.15s ease;
}

.btn-primary:hover:not(:disabled) {
  background: #1d4ed8;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

/* Success banner */
.success-banner {
  display: flex;
  justify-content: space-between;
  align-items: center;
  background: #d1fae5;
  color: #065f46;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

.close-btn {
  background: none;
  border: none;
  color: #065f46;
  font-size: 1.25rem;
  line-height: 1;
  cursor: pointer;
  padding: 0 0.25rem;
  flex-shrink: 0;
  margin-left: 1rem;
  opacity: 0.7;
  transition: opacity 0.15s ease;
}

.close-btn:hover {
  opacity: 1;
}

/* Empty state banner */
.empty-banner {
  background: #fef3c7;
  color: #92400e;
  padding: 1rem 1.25rem;
  border-radius: 8px;
  margin-bottom: 1.25rem;
  font-size: 0.938rem;
  font-weight: 500;
}

/* Muted row (zero recommended quantity) */
.muted {
  opacity: 0.5;
}

/* In-table loading state */
.table-loading {
  text-align: center;
  padding: 2rem;
  color: #64748b;
  font-size: 0.938rem;
}
</style>
