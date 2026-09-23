<script setup>
import { ref, computed } from 'vue'
import { useDietRecordStore } from '@/stores/dietRecord'
import { useMealPlanStore } from '@/stores/mealPlan'
import { DISH_CATEGORIES, MEALS, MEAL_ICONS, WEEK_DAYS } from '@/constants'
import { toDateKey, weekDateKeys, parseDateKey, formatDate } from '@/utils/date'
import { nutritionScore, scoreLabel } from '@/utils/nutrition'
import BaseButton from '@/components/common/BaseButton.vue'
import BaseTag from '@/components/common/BaseTag.vue'
import BaseEmpty from '@/components/common/BaseEmpty.vue'
import BaseModal from '@/components/common/BaseModal.vue'
import SimpleChart from '@/components/common/SimpleChart.vue'

const diet = useDietRecordStore()
const mealPlan = useMealPlanStore()

const date = ref(toDateKey())
const meal = ref('早餐')
const dishes = ref([{ name: '', category: '蔬菜' }])
const weekDates = weekDateKeys()

const dayRecords = computed(() => diet.records.filter((r) => r.date === date.value))
const dayDishes = computed(() => dayRecords.value.flatMap((r) => r.dishes))
const dayScore = computed(() => nutritionScore(dayDishes.value))
const score = computed(() => scoreLabel(dayScore.value))

const planDishesForToday = computed(() => {
  // 找到本周对应日期的计划菜品
  const idx = weekDates.indexOf(date.value)
  if (idx === -1) return []
  const dayKey = WEEK_DAYS[idx]?.key
  if (!dayKey) return []
  const week = mealPlan.currentWeek.days
  const m = { 早餐: 'breakfast', 午餐: 'lunch', 晚餐: 'dinner' }[meal.value]
  return (week[dayKey]?.[m] || [])
    .map((id) => mealPlan.dishMap[id])
    .filter(Boolean)
})

// 最近常吃候选：当前餐次优先，不足 8 道时用其他餐次补齐，并排除当日该餐次已记录的菜
const recentCandidates = computed(() => {
  const recordedNames = new Set(diet.mealDishes(date.value, meal.value).map((d) => d.name))
  const sameMeal = diet.recentDishes(meal.value, 8, date.value).filter((d) => !recordedNames.has(d.name))
  if (sameMeal.length >= 8) return sameMeal.slice(0, 8)
  const seen = new Set(sameMeal.map((d) => d.name))
  const others = diet
    .recentDishes('', 8, date.value)
    .filter((d) => !recordedNames.has(d.name) && !seen.has(d.name))
    .slice(0, 8 - sameMeal.length)
  return [...sameMeal, ...others]
})

const quickImportAvailable = computed(() => planDishesForToday.value.length > 0 || recentCandidates.value.length > 0)

const trendLabels = computed(() => weekDates.map((d) => `${parseDateKey(d).getMonth() + 1}/${parseDateKey(d).getDate()}`))
const trendData = computed(() => weekDates.map((d) => diet.dailyScores[d] || 0))

// 最近常吃选择弹窗
const showRecentPicker = ref(false)
const pickedKeys = ref([])

function addDish() {
  dishes.value.push({ name: '', category: '蔬菜' })
}
function removeDish(i) {
  if (dishes.value.length === 1) dishes.value[0] = { name: '', category: '蔬菜' }
  else dishes.value.splice(i, 1)
}

function candidateKey(d) {
  return `${d.name}|${d.category}`
}

function importFromPlan() {
  dishes.value = planDishesForToday.value.map((d) => ({ name: d.name, category: d.category }))
  if (!dishes.value.length) dishes.value = [{ name: '', category: '蔬菜' }]
}

// 无计划菜时一键带出最近常吃：默认勾选前 4 道
function openRecentPicker() {
  if (planDishesForToday.value.length) {
    importFromPlan()
    return
  }
  pickedKeys.value = recentCandidates.value.slice(0, 4).map(candidateKey)
  showRecentPicker.value = true
}

function togglePicked(d) {
  const key = candidateKey(d)
  const idx = pickedKeys.value.indexOf(key)
  if (idx === -1) pickedKeys.value.push(key)
  else pickedKeys.value.splice(idx, 1)
}

function applyRecentDishes() {
  const picked = recentCandidates.value.filter((d) => pickedKeys.value.includes(candidateKey(d)))
  if (picked.length) dishes.value = picked.map((d) => ({ name: d.name, category: d.category }))
  showRecentPicker.value = false
}

function save() {
  const valid = dishes.value.filter((d) => d.name.trim())
  if (!valid.length) return
  diet.addRecord(date.value, meal.value, valid)
  dishes.value = [{ name: '', category: '蔬菜' }]
}

function removeRecord(id) {
  diet.removeRecord(id)
}
</script>

<template>
  <div>
    <div class="page-head">
      <h2>🍽️ 每日饮食记录</h2>
    </div>

    <div class="card">
      <div class="section-title">记录一餐</div>
      <div class="form-grid">
        <div class="field">
          <label>日期</label>
          <input v-model="date" type="date" />
        </div>
        <div class="field">
          <label>餐次</label>
          <select v-model="meal">
            <option v-for="m in MEALS" :key="m" :value="m">{{ MEAL_ICONS[m] }} {{ m }}</option>
          </select>
        </div>
        <div class="field actions-col">
          <BaseButton variant="ghost" size="sm" :disabled="!quickImportAvailable" @click="openRecentPicker">
            {{ planDishesForToday.length ? '从计划导入' : '最近常吃' }}
          </BaseButton>
        </div>
      </div>

      <div class="dish-editor">
        <div v-for="(d, i) in dishes" :key="i" class="dish-row">
          <input v-model="d.name" type="text" placeholder="菜品名" class="grow" />
          <select v-model="d.category">
            <option v-for="c in DISH_CATEGORIES" :key="c" :value="c">{{ c }}</option>
          </select>
          <button class="del" @click="removeDish(i)">✕</button>
        </div>
        <div class="editor-actions">
          <BaseButton size="sm" variant="ghost" @click="addDish">+ 加一道菜</BaseButton>
          <BaseButton size="sm" @click="save">保存记录</BaseButton>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="section-title">
        <span>{{ date }} 记录</span>
        <BaseTag :text="`${dayScore} 分 · ${score.label}`" :color="score.color" />
      </div>
      <BaseEmpty v-if="!dayRecords.length" emoji="🍚" text="当天还没有记录" />
      <div v-else class="day-records">
        <div v-for="r in dayRecords" :key="r.id" class="rec">
          <div class="rec-head">
            <span class="meal">{{ MEAL_ICONS[r.meal] }} {{ r.meal }}</span>
            <button class="del" @click="removeRecord(r.id)">✕</button>
          </div>
          <div class="rec-dishes">
            <BaseTag v-for="(d, i) in r.dishes" :key="i" :category="d.category" :text="d.name" />
          </div>
        </div>
      </div>
    </div>

    <div class="card">
      <div class="section-title">本周营养评分趋势</div>
      <SimpleChart type="line" :labels="trendLabels" :data="trendData" color="#2196f3" :height="180" />
    </div>

    <!-- 无计划菜时，从最近常吃中挑选带入 -->
    <BaseModal :show="showRecentPicker" :title="`选择最近常吃的菜 · ${meal}`" @close="showRecentPicker = false">
      <BaseEmpty v-if="!recentCandidates.length" emoji="🍽️" text="还没有历史记录，先手动记录一餐吧" />
      <template v-else>
        <p class="picker-hint">默认已勾选最近常吃的几道菜，可点选增减</p>
        <div class="recent-grid">
          <button
            v-for="d in recentCandidates"
            :key="candidateKey(d)"
            type="button"
            class="recent-chip"
            :class="{ active: pickedKeys.includes(candidateKey(d)) }"
            @click="togglePicked(d)"
          >
            <span class="check">{{ pickedKeys.includes(candidateKey(d)) ? '✓' : '' }}</span>
            <span class="chip-name">{{ d.name }}</span>
            <BaseTag :category="d.category" :text="d.category" />
            <span class="chip-meta">吃过 {{ d.count }} 次 · {{ formatDate(d.lastDate) }}</span>
          </button>
        </div>
      </template>
      <template #footer>
        <BaseButton variant="ghost" size="sm" @click="showRecentPicker = false">取消</BaseButton>
        <BaseButton size="sm" :disabled="!pickedKeys.length" @click="applyRecentDishes">
          带入（{{ pickedKeys.length }}）
        </BaseButton>
      </template>
    </BaseModal>
  </div>
</template>

<style scoped>
.page-head h2 {
  margin: 0 0 16px;
}
.form-grid {
  display: grid;
  grid-template-columns: 1fr 1fr auto;
  gap: 12px;
  align-items: end;
  margin-bottom: 16px;
}
.field {
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.field label {
  font-size: 12px;
  color: var(--text-2);
}
.field input,
.field select {
  padding: 9px 12px;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 14px;
}
.actions-col {
  padding-bottom: 2px;
}
.dish-editor {
  border-top: 1px solid var(--border);
  padding-top: 16px;
}
.dish-row {
  display: flex;
  gap: 8px;
  margin-bottom: 8px;
}
.dish-row input {
  flex: 1;
  padding: 8px 12px;
  border: 1px solid var(--border);
  border-radius: 8px;
  font-size: 14px;
}
.dish-row .grow {
  flex: 2;
}
.dish-row select {
  width: 90px;
  border: 1px solid var(--border);
  border-radius: 8px;
}
.del {
  border: none;
  background: var(--danger-light);
  color: var(--danger);
  width: 32px;
  border-radius: 6px;
  cursor: pointer;
}
.editor-actions {
  display: flex;
  justify-content: space-between;
  margin-top: 8px;
}
.day-records {
  display: flex;
  flex-direction: column;
  gap: 12px;
}
.rec {
  border: 1px solid var(--border);
  border-radius: 8px;
  padding: 12px;
}
.rec-head {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 8px;
}
.meal {
  font-weight: 600;
}
.rec-dishes {
  display: flex;
  flex-wrap: wrap;
  gap: 6px;
}
.picker-hint {
  margin: 0 0 12px;
  font-size: 12px;
  color: var(--text-2);
}
.recent-grid {
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
}
.recent-chip {
  display: inline-flex;
  align-items: center;
  gap: 6px;
  padding: 6px 10px;
  border: 1px solid var(--border);
  border-radius: 8px;
  background: #fff;
  cursor: pointer;
  font-size: 13px;
  transition: all 0.15s;
}
.recent-chip:hover {
  border-color: var(--primary);
}
.recent-chip.active {
  border-color: var(--primary);
  background: var(--primary-light);
}
.recent-chip .check {
  display: inline-flex;
  align-items: center;
  justify-content: center;
  width: 16px;
  height: 16px;
  border-radius: 50%;
  border: 1px solid var(--border);
  font-size: 11px;
  color: #fff;
  flex-shrink: 0;
}
.recent-chip.active .check {
  background: var(--primary);
  border-color: var(--primary);
}
.chip-name {
  font-weight: 600;
}
.chip-meta {
  font-size: 11px;
  color: var(--text-2);
}
</style>
