<template>
  <div class="restocking">
    <div class="page-header">
      <h2>Restocking</h2>
      <p>Review recommended items based on demand forecasts and low stock levels.</p>
    </div>

    <div v-if="loading" class="loading">Loading...</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else>
      <!-- Budget bar card -->
      <div class="card budget-card">
        <div class="budget-header">
          <div>
            <div class="budget-label">Available Budget</div>
            <div class="budget-value">${{ budget.toLocaleString() }}</div>
          </div>
          <div class="budget-stats">
            <span>{{ selectedCount }} of {{ recommendations.length }} items selected</span>
            <span class="separator">·</span>
            <span>Est. total: ${{ selectedTotal.toLocaleString() }}</span>
          </div>
        </div>
        <input
          type="range"
          class="budget-slider"
          v-model.number="budget"
          :min="0"
          :max="totalCostWithData"
          :step="100"
        />
        <div class="budget-range-labels">
          <span>$0</span>
          <span>${{ totalCostWithData.toLocaleString() }}</span>
        </div>
      </div>

      <!-- Recommendations table -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">Recommended Items ({{ recommendations.length }})</h3>
        </div>
        <div class="table-container">
          <table class="restock-table">
            <thead>
              <tr>
                <th>Item</th>
                <th>SKU</th>
                <th>Source</th>
                <th>Current Stock</th>
                <th>Order Qty</th>
                <th>Unit Cost</th>
                <th>Total Cost</th>
                <th>Status</th>
              </tr>
            </thead>
            <tbody>
              <tr
                v-for="item in recommendations"
                :key="item.sku"
                :class="{ 'row-excluded': !isIncluded(item) }"
              >
                <td><strong>{{ item.name }}</strong></td>
                <td class="sku-cell">{{ item.sku }}</td>
                <td>
                  <span :class="['badge', sourceClass(item)]">{{ sourceLabel(item) }}</span>
                </td>
                <td>{{ item.quantity_on_hand !== undefined ? item.quantity_on_hand.toLocaleString() : '—' }}</td>
                <td>{{ item.quantity_to_order.toLocaleString() }}</td>
                <td>{{ item.has_cost ? '$' + item.unit_cost.toFixed(2) : 'N/A' }}</td>
                <td>{{ item.has_cost ? '$' + (item.quantity_to_order * item.unit_cost).toLocaleString() : '—' }}</td>
                <td>
                  <span v-if="isIncluded(item)" class="status-included">Included</span>
                  <span v-else-if="!item.has_cost" class="status-review">Review Required</span>
                  <span v-else class="status-excluded">Exceeds budget</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Place Order button -->
      <div class="order-footer">
        <div v-if="orderSuccess" class="order-success">
          Order {{ lastOrderNumber }} placed successfully. View it in the Orders tab.
        </div>
        <div v-if="orderError" class="order-error">{{ orderError }}</div>
        <button
          class="btn-primary"
          :disabled="selectedCount === 0 || placing"
          @click="placeOrder"
        >
          {{ placing ? 'Placing Order...' : `Place Order (${selectedCount} items)` }}
        </button>
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
    const { selectedLocation, selectedCategory, getCurrentFilters } = useFilters()

    // Raw data from APIs
    const demandForecasts = ref([])
    const inventoryItems = ref([])

    // UI state
    const loading = ref(true)
    const error = ref(null)
    const placing = ref(false)
    const orderSuccess = ref(false)
    const orderError = ref(null)
    const lastOrderNumber = ref(null)

    // Budget slider — initialised after data loads in loadData()
    const budget = ref(0)

    // ─── Step 1: O(1) inventory lookup by SKU ───────────────────────────────
    const inventoryBySku = computed(() => {
      const map = {}
      inventoryItems.value.forEach(item => { map[item.sku] = item })
      return map
    })

    // ─── Step 2: Demand-based recommendations (non-decreasing trends) ────────
    const demandRecommendations = computed(() => {
      return demandForecasts.value
        .filter(forecast => forecast.trend !== 'decreasing')
        .map(forecast => {
          const inv = inventoryBySku.value[forecast.item_sku]

          let quantity_to_order
          let unit_cost
          let has_cost
          let quantity_on_hand

          if (inv) {
            // Inventory match found
            quantity_on_hand = inv.quantity_on_hand
            unit_cost = inv.unit_cost
            has_cost = true

            if (forecast.forecasted_demand > inv.quantity_on_hand) {
              // Stock will not cover forecasted demand — order the shortfall
              quantity_to_order = forecast.forecasted_demand - inv.quantity_on_hand
            } else if (forecast.trend === 'increasing') {
              // Increasing trend but stock is currently sufficient — proactive 20% buffer
              quantity_to_order = Math.ceil(forecast.forecasted_demand * 0.2)
            } else {
              // Stable trend, stock sufficient — small proactive buffer
              quantity_to_order = Math.ceil(forecast.forecasted_demand * 0.2)
            }
          } else {
            // No inventory match — no cost data available
            quantity_on_hand = undefined
            unit_cost = 0
            has_cost = false
            quantity_to_order = Math.ceil(forecast.forecasted_demand * 0.3)
          }

          return {
            sku: forecast.item_sku,
            name: forecast.item_name,
            trend: forecast.trend,
            source: 'demand',
            quantity_on_hand,
            quantity_to_order,
            unit_cost,
            has_cost
          }
        })
    })

    // ─── Step 3: Below-reorder-point items not already covered by demand ─────
    const reorderRecommendations = computed(() => {
      // Track which SKUs are already covered by demand recommendations
      const demandSkus = new Set(demandRecommendations.value.map(r => r.sku))

      return inventoryItems.value
        .filter(item => item.quantity_on_hand < item.reorder_point)
        .filter(item => !demandSkus.has(item.sku))
        .map(item => {
          // Order enough to reach reorder_point, plus a 50% safety buffer on top
          const quantity_to_order =
            (item.reorder_point - item.quantity_on_hand) +
            Math.ceil(item.reorder_point * 0.5)

          return {
            sku: item.sku,
            name: item.name,
            trend: null,
            source: 'reorder',
            quantity_on_hand: item.quantity_on_hand,
            quantity_to_order,
            unit_cost: item.unit_cost,
            has_cost: true
          }
        })
    })

    // ─── Step 4: Merged and sorted recommendations ───────────────────────────
    // Priority order:
    //   1. increasing demand items (highest urgency)
    //   2. below-reorder-point items
    //   3. stable demand items
    const recommendations = computed(() => {
      const all = [...demandRecommendations.value, ...reorderRecommendations.value]

      return all.slice().sort((a, b) => {
        const priority = (item) => {
          if (item.trend === 'increasing') return 0
          if (item.source === 'reorder') return 1
          return 2 // stable demand
        }
        return priority(a) - priority(b)
      })
    })

    // ─── Step 5: Budget computations ─────────────────────────────────────────

    // Sum of item costs for items where cost is known
    const totalCostWithData = computed(() => {
      return recommendations.value
        .filter(item => item.has_cost)
        .reduce((sum, item) => sum + item.quantity_to_order * item.unit_cost, 0)
    })

    // Greedy budget selection:
    // Items without cost data are ALWAYS included (unknown cost, don't consume budget).
    // Remaining items are processed in priority order; we accumulate a running total
    // and include an item only while the running total stays within the budget.
    const withinBudget = computed(() => {
      const selected = new Set()
      let running = 0

      for (const item of recommendations.value) {
        if (!item.has_cost) {
          // No cost — always include, does not consume budget
          selected.add(item.sku)
          continue
        }

        const itemCost = item.quantity_to_order * item.unit_cost
        if (running + itemCost <= budget.value) {
          selected.add(item.sku)
          running += itemCost
        }
        // Once budget is exceeded we keep iterating in case a cheaper item fits,
        // but the sorted priority order means earlier items are always preferred.
      }

      return selected
    })

    // ─── Derived UI stats ─────────────────────────────────────────────────────

    const selectedCount = computed(() => {
      return recommendations.value.filter(item => isIncluded(item)).length
    })

    const selectedTotal = computed(() => {
      return recommendations.value
        .filter(item => isIncluded(item) && item.has_cost)
        .reduce((sum, item) => sum + item.quantity_to_order * item.unit_cost, 0)
    })

    // ─── Helpers ──────────────────────────────────────────────────────────────

    const isIncluded = (item) => {
      return !item.has_cost || withinBudget.value.has(item.sku)
    }

    const sourceLabel = (item) => {
      if (item.source === 'demand' && item.trend === 'increasing') return 'Increasing'
      if (item.source === 'demand') return 'Forecast'
      return 'Low Stock'
    }

    const sourceClass = (item) => {
      if (item.source === 'reorder') return 'badge-reorder'
      if (item.trend === 'increasing') return 'badge-demand'
      return 'badge-demand'
    }

    // ─── Data loading ─────────────────────────────────────────────────────────

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()

        const [forecasts, inventory] = await Promise.all([
          api.getDemandForecasts(),
          api.getInventory({ warehouse: filters.warehouse, category: filters.category })
        ])

        demandForecasts.value = forecasts
        inventoryItems.value = inventory

        // Initialise the budget slider to the full cost so all items are selected by default
        budget.value = totalCostWithData.value
      } catch (err) {
        error.value = 'Failed to load restocking data: ' + err.message
        console.error(err)
      } finally {
        loading.value = false
      }
    }

    // Reload inventory (and re-derive recommendations) when location/category changes.
    // Demand forecasts are filter-agnostic so we reload both for simplicity.
    watch([selectedLocation, selectedCategory], () => {
      loadData()
    })

    onMounted(loadData)

    // ─── Place order ──────────────────────────────────────────────────────────

    const placeOrder = async () => {
      orderSuccess.value = false
      orderError.value = null
      placing.value = true

      try {
        const items = recommendations.value
          .filter(item => isIncluded(item))
          .map(item => ({
            sku: item.sku,
            name: item.name,
            quantity: item.quantity_to_order,
            unit_cost: item.unit_cost
          }))

        const order = await api.createRestockingOrder(items)
        lastOrderNumber.value = order.order_number
        orderSuccess.value = true
      } catch (err) {
        orderError.value = 'Failed to place order: ' + err.message
        console.error(err)
      } finally {
        placing.value = false
      }
    }

    return {
      loading,
      error,
      budget,
      totalCostWithData,
      recommendations,
      selectedCount,
      selectedTotal,
      withinBudget,
      placing,
      orderSuccess,
      orderError,
      lastOrderNumber,
      isIncluded,
      sourceLabel,
      sourceClass,
      placeOrder
    }
  }
}
</script>

<style scoped>
/* ── Budget card ──────────────────────────────────────────────────────────── */
.budget-card { margin-bottom: 0; }

.budget-header {
  display: flex;
  justify-content: space-between;
  align-items: flex-start;
  margin-bottom: 16px;
}

.budget-label {
  font-size: 12px;
  font-weight: 600;
  text-transform: uppercase;
  letter-spacing: .06em;
  color: #64748b;
}

.budget-value {
  font-size: 28px;
  font-weight: 700;
  color: #0f172a;
  margin-top: 4px;
}

.budget-stats {
  font-size: 13px;
  color: #64748b;
  text-align: right;
  display: flex;
  gap: 8px;
  align-items: center;
}

.separator { color: #cbd5e1; }

.budget-slider {
  width: 100%;
  height: 6px;
  accent-color: #2563eb;
  cursor: pointer;
}

.budget-range-labels {
  display: flex;
  justify-content: space-between;
  font-size: 11px;
  color: #94a3b8;
  margin-top: 4px;
}

/* ── Table ────────────────────────────────────────────────────────────────── */
.restock-table {
  width: 100%;
  border-collapse: collapse;
}

.restock-table th,
.restock-table td {
  padding: 10px 12px;
  text-align: left;
  border-bottom: 1px solid #f1f5f9;
  font-size: 13px;
}

.restock-table th {
  font-size: 11px;
  font-weight: 700;
  text-transform: uppercase;
  letter-spacing: .06em;
  color: #64748b;
  background: #f8fafc;
}

.sku-cell {
  font-family: monospace;
  font-size: 12px;
  color: #475569;
}

.row-excluded { opacity: 0.4; }

/* ── Source badges ────────────────────────────────────────────────────────── */
.badge-demand  { background: #dbeafe; color: #1d4ed8; }
.badge-reorder { background: #fef3c7; color: #b45309; }
.badge-both    { background: #ede9fe; color: #6d28d9; }

/* ── Status labels ────────────────────────────────────────────────────────── */
.status-included { color: #16a34a; font-weight: 600; font-size: 12px; }
.status-excluded { color: #94a3b8; font-size: 12px; }
.status-review   { color: #d97706; font-weight: 600; font-size: 12px; }

/* ── Order footer ─────────────────────────────────────────────────────────── */
.order-footer {
  display: flex;
  justify-content: flex-end;
  align-items: center;
  gap: 16px;
  padding: 8px 0;
}

.btn-primary {
  background: #2563eb;
  color: white;
  border: none;
  padding: 10px 24px;
  border-radius: 8px;
  font-size: 14px;
  font-weight: 600;
  cursor: pointer;
}

.btn-primary:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.btn-primary:not(:disabled):hover { background: #1d4ed8; }

.order-success {
  color: #16a34a;
  font-size: 13px;
  font-weight: 600;
}

.order-error {
  color: #dc2626;
  font-size: 13px;
}
</style>
