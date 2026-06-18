<script setup lang="ts">
import { ref, computed } from 'vue'
import { useGameStore } from '../stores/gameStore'
import { getTimeLabel } from '../utils/gameUtils'
import type { ResourceChangeCategory } from '../types/game'

const emit = defineEmits<{
  (e: 'close'): void
}>()

const gameStore = useGameStore()
const selectedDay = ref<number>(gameStore.day)

const availableDays = computed(() => {
  const days = new Set<number>()
  days.add(gameStore.day)
  gameStore.daysWithResourceChanges.forEach(d => days.add(d))
  for (let i = 1; i <= gameStore.day; i++) {
    days.add(i)
  }
  return Array.from(days).sort((a, b) => b - a)
})

const currentSummary = computed(() => {
  return gameStore.getDaySummary(selectedDay.value)
})

const categoryIcons: Record<ResourceChangeCategory, string> = {
  work: '💼',
  gift: '🎁',
  event: '📖',
  system: '⚙️'
}

const categoryColors: Record<ResourceChangeCategory, string> = {
  work: 'var(--success-color)',
  gift: 'var(--accent-primary)',
  event: 'var(--info-color)',
  system: 'var(--warning-color)'
}

function formatNumber(num: number): string {
  if (num > 0) return `+${num}`
  return num.toString()
}
</script>

<template>
  <Teleport to="body">
    <div class="modal-overlay" @click.self="emit('close')">
      <div class="modal-content review-modal">
        <div class="modal-header">
          <h2>📊 按天复盘</h2>
          <button class="close-btn" @click="emit('close')">✕</button>
        </div>

        <div class="day-selector">
          <button
            v-for="day in availableDays"
            :key="day"
            class="day-btn"
            :class="{ active: selectedDay === day, current: day === gameStore.day }"
            @click="selectedDay = day"
          >
            第 {{ day }} 天
            <span v-if="day === gameStore.day" class="current-badge">今天</span>
          </button>
        </div>

        <div class="summary-card">
          <div class="summary-header">
            <h3>第 {{ selectedDay }} 天 代币复盘</h3>
            <div class="summary-balance">
              <span class="balance-label">当前余额</span>
              <span class="balance-value">💰 {{ gameStore.resources }}</span>
            </div>
          </div>

          <div class="summary-stats">
            <div class="stat-item income">
              <span class="stat-icon">📈</span>
              <div class="stat-info">
                <span class="stat-label">今日收入</span>
                <span class="stat-value">+{{ currentSummary.totalIncome }}</span>
              </div>
            </div>
            <div class="stat-item expense">
              <span class="stat-icon">📉</span>
              <div class="stat-info">
                <span class="stat-label">今日支出</span>
                <span class="stat-value">-{{ currentSummary.totalExpense }}</span>
              </div>
            </div>
            <div class="stat-item net" :class="{ positive: currentSummary.netChange >= 0, negative: currentSummary.netChange < 0 }">
              <span class="stat-icon">{{ currentSummary.netChange >= 0 ? '📊' : '📉' }}</span>
              <div class="stat-info">
                <span class="stat-label">今日净变化</span>
                <span class="stat-value">{{ formatNumber(currentSummary.netChange) }}</span>
              </div>
            </div>
          </div>

          <div class="balance-flow">
            <div class="flow-item">
              <span class="flow-label">日初余额</span>
              <span class="flow-value">{{ currentSummary.dayStartResources }}</span>
            </div>
            <span class="flow-arrow">→</span>
            <div class="flow-item">
              <span class="flow-label">收入</span>
              <span class="flow-value income">+{{ currentSummary.totalIncome }}</span>
            </div>
            <span class="flow-arrow">→</span>
            <div class="flow-item">
              <span class="flow-label">支出</span>
              <span class="flow-value expense">-{{ currentSummary.totalExpense }}</span>
            </div>
            <span class="flow-arrow">=</span>
            <div class="flow-item final">
              <span class="flow-label">日末余额</span>
              <span class="flow-value">{{ currentSummary.dayEndResources }}</span>
            </div>
          </div>
        </div>

        <div class="details-section">
          <div class="details-column">
            <div class="column-header income-header">
              <span>📈 收入来源</span>
            </div>

            <div v-if="currentSummary.incomeDetails.length === 0" class="empty-state">
              今日暂无收入
            </div>

            <div v-else class="category-group">
              <div
                v-for="(amount, category) in currentSummary.incomeByCategory"
                :key="category"
                class="category-item"
              >
                <span class="category-icon">{{ categoryIcons[category as ResourceChangeCategory] }}</span>
                <span class="category-name">{{ gameStore.getCategoryLabel(category as ResourceChangeCategory) }}</span>
                <span class="category-amount income">+{{ amount }}</span>
              </div>
            </div>

            <div v-if="currentSummary.incomeDetails.length > 0" class="detail-list">
              <div
                v-for="item in currentSummary.incomeDetails"
                :key="item.id"
                class="detail-item animate-fade-in"
              >
                <div class="detail-icon">{{ categoryIcons[item.category] }}</div>
                <div class="detail-info">
                  <span class="detail-desc">{{ item.description }}</span>
                  <span class="detail-time">{{ getTimeLabel(item.time) }}</span>
                </div>
                <span class="detail-amount income">+{{ item.amount }}</span>
              </div>
            </div>
          </div>

          <div class="details-column">
            <div class="column-header expense-header">
              <span>📉 支出去向</span>
            </div>

            <div v-if="currentSummary.expenseDetails.length === 0" class="empty-state">
              今日暂无支出
            </div>

            <div v-else class="category-group">
              <div
                v-for="(amount, category) in currentSummary.expenseByCategory"
                :key="category"
                class="category-item"
              >
                <span class="category-icon">{{ categoryIcons[category as ResourceChangeCategory] }}</span>
                <span class="category-name">{{ gameStore.getCategoryLabel(category as ResourceChangeCategory) }}</span>
                <span class="category-amount expense">-{{ amount }}</span>
              </div>
            </div>

            <div v-if="currentSummary.expenseDetails.length > 0" class="detail-list">
              <div
                v-for="item in currentSummary.expenseDetails"
                :key="item.id"
                class="detail-item animate-fade-in"
              >
                <div class="detail-icon">{{ categoryIcons[item.category] }}</div>
                <div class="detail-info">
                  <span class="detail-desc">{{ item.description }}</span>
                  <span class="detail-time">{{ getTimeLabel(item.time) }}</span>
                </div>
                <span class="detail-amount expense">-{{ item.amount }}</span>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>
  </Teleport>
</template>

<style scoped>
.review-modal {
  padding: 24px;
  max-width: 800px;
  width: 95%;
}

.modal-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.modal-header h2 {
  font-size: 20px;
  font-weight: 600;
}

.close-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  background: var(--bg-tertiary);
  font-size: 16px;
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-btn:hover {
  background: var(--accent-light);
}

.day-selector {
  display: flex;
  gap: 8px;
  overflow-x: auto;
  padding-bottom: 12px;
  margin-bottom: 16px;
  scrollbar-width: thin;
}

.day-selector::-webkit-scrollbar {
  height: 6px;
}

.day-selector::-webkit-scrollbar-track {
  background: var(--bg-tertiary);
  border-radius: 3px;
}

.day-selector::-webkit-scrollbar-thumb {
  background: var(--border-color);
  border-radius: 3px;
}

.day-btn {
  display: flex;
  align-items: center;
  gap: 6px;
  padding: 8px 16px;
  border-radius: var(--radius-md);
  background: var(--bg-tertiary);
  font-size: 14px;
  font-weight: 500;
  white-space: nowrap;
  flex-shrink: 0;
}

.day-btn:hover {
  background: var(--accent-light);
}

.day-btn.active {
  background: linear-gradient(135deg, var(--accent-primary), var(--accent-secondary));
  color: white;
}

.day-btn.current {
  border: 2px solid var(--accent-primary);
}

.day-btn.active.current {
  border-color: white;
}

.current-badge {
  font-size: 11px;
  padding: 2px 6px;
  border-radius: 4px;
  background: rgba(255, 255, 255, 0.2);
}

.day-btn:not(.active) .current-badge {
  background: var(--accent-light);
  color: var(--accent-primary);
}

.summary-card {
  background: var(--bg-tertiary);
  border-radius: var(--radius-lg);
  padding: 20px;
  margin-bottom: 20px;
}

.summary-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.summary-header h3 {
  font-size: 18px;
  font-weight: 600;
}

.summary-balance {
  display: flex;
  flex-direction: column;
  align-items: flex-end;
  gap: 4px;
}

.balance-label {
  font-size: 12px;
  color: var(--text-secondary);
}

.balance-value {
  font-size: 20px;
  font-weight: 700;
  color: var(--accent-primary);
}

.summary-stats {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
  margin-bottom: 16px;
}

.stat-item {
  display: flex;
  align-items: center;
  gap: 12px;
  padding: 16px;
  background: var(--bg-secondary);
  border-radius: var(--radius-md);
}

.stat-icon {
  font-size: 28px;
}

.stat-info {
  display: flex;
  flex-direction: column;
  gap: 4px;
}

.stat-label {
  font-size: 12px;
  color: var(--text-secondary);
}

.stat-value {
  font-size: 20px;
  font-weight: 700;
}

.stat-item.income .stat-value {
  color: var(--success-color);
}

.stat-item.expense .stat-value {
  color: var(--error-color);
}

.stat-item.net.positive .stat-value {
  color: var(--success-color);
}

.stat-item.net.negative .stat-value {
  color: var(--error-color);
}

.balance-flow {
  display: flex;
  align-items: center;
  justify-content: center;
  gap: 8px;
  padding: 12px;
  background: var(--bg-secondary);
  border-radius: var(--radius-md);
  flex-wrap: wrap;
}

.flow-item {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 4px;
}

.flow-label {
  font-size: 11px;
  color: var(--text-muted);
}

.flow-value {
  font-size: 14px;
  font-weight: 600;
}

.flow-value.income {
  color: var(--success-color);
}

.flow-value.expense {
  color: var(--error-color);
}

.flow-item.final .flow-value {
  color: var(--accent-primary);
  font-size: 16px;
}

.flow-arrow {
  font-size: 14px;
  color: var(--text-muted);
}

.details-section {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 16px;
}

@media (max-width: 700px) {
  .details-section {
    grid-template-columns: 1fr;
  }
}

.details-column {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.column-header {
  padding: 10px 14px;
  border-radius: var(--radius-md);
  font-weight: 600;
  font-size: 14px;
}

.income-header {
  background: #dcfce7;
  color: #166534;
}

[data-theme='dark'] .income-header {
  background: #14532d;
  color: #86efac;
}

.expense-header {
  background: #fee2e2;
  color: #991b1b;
}

[data-theme='dark'] .expense-header {
  background: #7f1d1d;
  color: #fca5a5;
}

.empty-state {
  text-align: center;
  padding: 30px;
  color: var(--text-muted);
  font-size: 14px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
}

.category-group {
  display: flex;
  flex-direction: column;
  gap: 8px;
  margin-bottom: 8px;
}

.category-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 14px;
  background: var(--bg-tertiary);
  border-radius: var(--radius-md);
}

.category-icon {
  font-size: 18px;
}

.category-name {
  flex: 1;
  font-size: 13px;
  font-weight: 500;
}

.category-amount {
  font-weight: 600;
  font-size: 14px;
}

.category-amount.income {
  color: var(--success-color);
}

.category-amount.expense {
  color: var(--error-color);
}

.detail-list {
  display: flex;
  flex-direction: column;
  gap: 6px;
  max-height: 250px;
  overflow-y: auto;
  padding-right: 4px;
}

.detail-item {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 10px 12px;
  background: var(--bg-secondary);
  border-radius: var(--radius-sm);
  border-left: 3px solid var(--border-color);
}

.detail-icon {
  font-size: 16px;
  flex-shrink: 0;
}

.detail-info {
  flex: 1;
  min-width: 0;
  display: flex;
  flex-direction: column;
  gap: 2px;
}

.detail-desc {
  font-size: 13px;
  overflow: hidden;
  text-overflow: ellipsis;
  white-space: nowrap;
}

.detail-time {
  font-size: 11px;
  color: var(--text-muted);
}

.detail-amount {
  font-weight: 600;
  font-size: 14px;
  flex-shrink: 0;
}

.detail-amount.income {
  color: var(--success-color);
}

.detail-amount.expense {
  color: var(--error-color);
}

@media (max-width: 700px) {
  .summary-stats {
    grid-template-columns: 1fr;
  }
  
  .balance-flow {
    flex-direction: column;
  }
  
  .flow-arrow {
    transform: rotate(90deg);
  }
}
</style>
