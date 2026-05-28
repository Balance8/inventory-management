<template>
  <div class="reports">
    <div class="page-header">
      <h2>{{ t('reports.title') }}</h2>
      <p>{{ t('reports.description') }}</p>
    </div>

    <div v-if="loading" class="loading">{{ t('common.loading') }}</div>
    <div v-else-if="error" class="error">{{ error }}</div>
    <div v-else-if="quarterlyData.length === 0 && monthlyData.length === 0">
      <div class="card">
        <p class="no-data-message">{{ t('reports.noData') }}</p>
      </div>
    </div>
    <div v-else>
      <!-- Quarterly Performance -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.quarterlyTitle') }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t('reports.table.quarter') }}</th>
                <th>{{ t('reports.table.totalOrders') }}</th>
                <th>{{ t('reports.table.totalRevenue') }}</th>
                <th>{{ t('reports.table.avgOrderValue') }}</th>
                <th>{{ t('reports.table.fulfillmentRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="q in quarterlyData" :key="q.quarter">
                <td><strong>{{ q.quarter }}</strong></td>
                <td>{{ formatNumber(q.total_orders) }}</td>
                <td>{{ formatRevenue(q.total_revenue) }}</td>
                <td>{{ formatRevenue(q.avg_order_value) }}</td>
                <td>
                  <span :class="getFulfillmentClass(q.fulfillment_rate)">
                    {{ q.fulfillment_rate }}%
                  </span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Monthly Revenue Trend Chart -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monthlyChartTitle') }}</h3>
        </div>
        <div class="chart-container">
          <div class="bar-chart">
            <div v-for="month in monthlyData" :key="month.month" class="bar-wrapper">
              <div class="bar-container">
                <div
                  class="bar"
                  :style="{ height: (maxMonthlyRevenue > 0 ? (month.revenue / maxMonthlyRevenue) * 200 : 0) + 'px' }"
                  :title="formatRevenue(month.revenue)"
                ></div>
              </div>
              <div class="bar-label">{{ formatMonth(month.month) }}</div>
            </div>
          </div>
        </div>
      </div>

      <!-- Month-over-Month Analysis -->
      <div class="card">
        <div class="card-header">
          <h3 class="card-title">{{ t('reports.monthOverMonthTitle') }}</h3>
        </div>
        <div class="table-container">
          <table class="reports-table">
            <thead>
              <tr>
                <th>{{ t('reports.table.month') }}</th>
                <th>{{ t('reports.table.orders') }}</th>
                <th>{{ t('reports.table.revenue') }}</th>
                <th>{{ t('reports.table.change') }}</th>
                <th>{{ t('reports.table.growthRate') }}</th>
              </tr>
            </thead>
            <tbody>
              <tr v-for="row in monthlyComparisons" :key="row.month">
                <td><strong>{{ formatMonth(row.month) }}</strong></td>
                <td>{{ formatNumber(row.orders) }}</td>
                <td>{{ formatRevenue(row.revenue) }}</td>
                <td>
                  <span v-if="row.isFirst">-</span>
                  <span v-else :class="row.changeClass">{{ row.changeDisplay }}</span>
                </td>
                <td>
                  <span v-if="row.isFirst">-</span>
                  <span v-else :class="row.changeClass">{{ row.growthDisplay }}</span>
                </td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>

      <!-- Summary Stats -->
      <div class="stats-grid">
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.summary.totalRevenueYTD') }}</div>
          <div class="stat-value">{{ totalRevenue > 0 ? formatRevenue(totalRevenue) : t('reports.notAvailable') }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.summary.avgMonthlyRevenue') }}</div>
          <div class="stat-value">{{ avgMonthlyRevenue > 0 ? formatRevenue(avgMonthlyRevenue) : t('reports.notAvailable') }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.summary.totalOrdersYTD') }}</div>
          <div class="stat-value">{{ totalOrders > 0 ? formatNumber(totalOrders) : '—' }}</div>
        </div>
        <div class="stat-card">
          <div class="stat-label">{{ t('reports.summary.bestQuarter') }}</div>
          <div class="stat-value">{{ bestQuarter || t('reports.notAvailable') }}</div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { ref, computed, onMounted, watch } from 'vue'
import { api } from '../api'
import { useFilters } from '../composables/useFilters'
import { useI18n } from '../composables/useI18n'
import { formatCurrency } from '../utils/currency'

export default {
  name: 'Reports',
  setup() {
    const { t, currentCurrency, currentLocale } = useI18n()
    const {
      selectedPeriod,
      selectedLocation,
      selectedCategory,
      selectedStatus,
      getCurrentFilters
    } = useFilters()

    const loading = ref(true)
    const error = ref(null)
    const quarterlyData = ref([])
    const monthlyData = ref([])

    const loadData = async () => {
      loading.value = true
      error.value = null
      try {
        const filters = getCurrentFilters()
        const [quarterly, monthly] = await Promise.all([
          api.getQuarterlyReports(filters),
          api.getMonthlyTrends(filters)
        ])
        quarterlyData.value = quarterly
        monthlyData.value = monthly
      } catch (err) {
        console.error('Failed to load reports:', err)
        error.value = 'Failed to load reports: ' + err.message
      } finally {
        loading.value = false
      }
    }

    watch([selectedPeriod, selectedLocation, selectedCategory, selectedStatus], () => {
      loadData()
    })

    onMounted(loadData)

    // --- Computed summary stats ---

    const totalRevenue = computed(() =>
      monthlyData.value.reduce((sum, m) => sum + (m.revenue || 0), 0)
    )

    const avgMonthlyRevenue = computed(() =>
      monthlyData.value.length > 0 ? totalRevenue.value / monthlyData.value.length : 0
    )

    const totalOrders = computed(() =>
      monthlyData.value.reduce((sum, m) => sum + (m.order_count || 0), 0)
    )

    const bestQuarter = computed(() => {
      if (quarterlyData.value.length === 0) return ''
      return quarterlyData.value.reduce(
        (best, q) => (q.total_revenue > best.total_revenue ? q : best),
        quarterlyData.value[0]
      ).quarter
    })

    const maxMonthlyRevenue = computed(() => {
      if (monthlyData.value.length === 0) return 0
      return Math.max(...monthlyData.value.map(m => m.revenue || 0))
    })

    // Pre-baked per-row comparisons so the template never does index arithmetic
    const monthlyComparisons = computed(() =>
      monthlyData.value.map((m, i) => {
        const isFirst = i === 0
        const prev = isFirst ? null : monthlyData.value[i - 1]
        return {
          month: m.month,
          orders: m.order_count,
          revenue: m.revenue,
          isFirst,
          changeDisplay: isFirst ? '' : getChangeValue(m.revenue, prev.revenue),
          changeClass: isFirst ? '' : getChangeClass(m.revenue, prev.revenue),
          growthDisplay: isFirst ? '' : getGrowthRate(m.revenue, prev.revenue)
        }
      })
    )

    // --- Helpers ---

    const formatNumber = (num) =>
      (num ?? 0).toLocaleString('en-US', { maximumFractionDigits: 0 })

    const formatRevenue = (num) =>
      formatCurrency(num ?? 0, currentCurrency.value)

    const MONTH_KEYS = ['jan', 'feb', 'mar', 'apr', 'may', 'jun', 'jul', 'aug', 'sep', 'oct', 'nov', 'dec']

    const formatMonth = (monthStr) => {
      if (!monthStr || typeof monthStr !== 'string') return monthStr
      const parts = monthStr.split('-')
      if (parts.length < 2) return monthStr
      const year = parts[0]
      const monthIndex = parseInt(parts[1], 10) - 1
      if (isNaN(monthIndex) || monthIndex < 0 || monthIndex > 11) return monthStr
      return t(`months.${MONTH_KEYS[monthIndex]}`) + ' ' + year
    }

    const getFulfillmentClass = (rate) => {
      if (rate >= 90) return 'badge success'
      if (rate >= 75) return 'badge warning'
      return 'badge danger'
    }

    const getChangeValue = (current, previous) => {
      const change = (current || 0) - (previous || 0)
      if (change > 0) return '+' + formatRevenue(change)
      if (change < 0) return '-' + formatRevenue(Math.abs(change))
      return formatRevenue(0)
    }

    const getChangeClass = (current, previous) => {
      const change = (current || 0) - (previous || 0)
      if (change > 0) return 'positive-change'
      if (change < 0) return 'negative-change'
      return ''
    }

    const getGrowthRate = (current, previous) => {
      if (!previous || previous === 0) return t('reports.notAvailable')
      const rate = ((current - previous) / previous) * 100
      const sign = rate > 0 ? '+' : ''
      return sign + rate.toFixed(1) + '%'
    }

    return {
      t,
      selectedCurrency: currentCurrency,
      currentLocale,
      loading,
      error,
      quarterlyData,
      monthlyData,
      totalRevenue,
      avgMonthlyRevenue,
      totalOrders,
      bestQuarter,
      maxMonthlyRevenue,
      monthlyComparisons,
      formatNumber,
      formatRevenue,
      formatMonth,
      getFulfillmentClass,
      getChangeValue,
      getChangeClass,
      getGrowthRate
    }
  }
}
</script>

<style scoped>
.reports {
  padding: 0;
}

.no-data-message {
  padding: 2rem;
  text-align: center;
  color: #64748b;
  margin: 0;
}

.reports-table {
  width: 100%;
  border-collapse: collapse;
}

.reports-table th {
  background: #f8fafc;
  padding: 0.75rem;
  text-align: left;
  font-weight: 600;
  color: #64748b;
  border-bottom: 2px solid #e2e8f0;
}

.reports-table td {
  padding: 0.75rem;
  border-bottom: 1px solid #e2e8f0;
}

.reports-table tr:hover {
  background: #f8fafc;
}

.chart-container {
  padding: 2rem 1rem;
  min-height: 300px;
}

.bar-chart {
  display: flex;
  align-items: flex-end;
  justify-content: space-around;
  height: 250px;
  gap: 0.5rem;
}

.bar-wrapper {
  display: flex;
  flex-direction: column;
  align-items: center;
  flex: 1;
  max-width: 80px;
}

.bar-container {
  height: 200px;
  display: flex;
  align-items: flex-end;
  width: 100%;
}

.bar {
  width: 100%;
  background: linear-gradient(to top, #3b82f6, #60a5fa);
  border-radius: 4px 4px 0 0;
  transition: all 0.3s;
  cursor: pointer;
}

.bar:hover {
  background: linear-gradient(to top, #2563eb, #3b82f6);
}

.bar-label {
  margin-top: 1.5rem;
  font-size: 0.75rem;
  color: #64748b;
  text-align: center;
  transform: rotate(-45deg);
  white-space: nowrap;
}

.positive-change {
  color: #16a34a;
  font-weight: 600;
}

.negative-change {
  color: #dc2626;
  font-weight: 600;
}
</style>
