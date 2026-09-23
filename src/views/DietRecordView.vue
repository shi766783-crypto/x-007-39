<script setup>
import { ref, computed, watch } from 'vue'
import { useDietRecordStore } from '@/stores/dietRecord'
import { useMealPlanStore } from '@/stores/mealPlan'
import { DISH_CATEGORIES, MEALS, MEAL_ICONS, WEEK_DAYS } from '@/constants'
import { toDateKey, weekDateKeys, parseDateKey } from '@/utils/date'
import { nutritionScore, scoreLabel } from '@/utils/nutrition'
import BaseButton from '@/components/common/BaseButton.vue'
import BaseTag from '@/components/common/BaseTag.vue'
import BaseEmpty from '@/components/common/BaseEmpty.vue'
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

// 当天该餐次没有计划菜时，提供最近常吃的菜品供快速复用
const recentDishes = computed(() => (planDishesForToday.value.length ? [] : diet.recentDishes(8)))
const showRecent = ref(false)
const addedNames = computed(() => new Set(dishes.value.map((d) => d.name.trim()).filter(Boolean)))

// 切换日期/餐次后收起常吃面板，避免与当前选择错位
watch([date, meal], () => {
  showRecent.value = false
})

const trendLabels = computed(() => weekDates.map((d) => `${parseDateKey(d).getMonth() + 1}/${parseDateKey(d).getDate()}`))
const trendData = computed(() => weekDates.map((d) => diet.dailyScores[d] || 0))

function addDish() {
  dishes.value.push({ name: '', category: '蔬菜' })
}
function removeDish(i) {
  if (dishes.value.length === 1) dishes.value[0] = { name: '', category: '蔬菜' }
  else dishes.value.splice(i, 1)
}

function importFromPlan() {
  dishes.value = planDishesForToday.value.map((d) => ({ name: d.name, category: d.category }))
  if (!dishes.value.length) dishes.value = [{ name: '', category: '蔬菜' }]
}

// 把最近常吃的菜加入编辑区：优先填充空行，已在编辑区的不重复添加
function addRecentDish(d) {
  if (addedNames.value.has(d.name)) return
  const empty = dishes.value.find((x) => !x.name.trim())
  if (empty) {
    empty.name = d.name
    empty.category = d.category
  } else {
    dishes.value.push({ name: d.name, category: d.category })
  }
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
          <BaseButton v-if="planDishesForToday.length" variant="ghost" size="sm" @click="importFromPlan">
            从计划导入
          </BaseButton>
          <BaseButton
            v-else-if="recentDishes.length"
            variant="ghost"
            size="sm"
            @click="showRecent = !showRecent"
          >
            🕘 最近常吃
          </BaseButton>
          <BaseButton v-else variant="ghost" size="sm" disabled>从计划导入</BaseButton>
        </div>
      </div>

      <div v-if="showRecent && recentDishes.length" class="recent-panel">
        <span class="recent-label">最近常吃（点击加入）：</span>
        <button
          v-for="d in recentDishes"
          :key="d.name"
          class="recent-chip"
          :class="{ added: addedNames.has(d.name) }"
          :disabled="addedNames.has(d.name)"
          @click="addRecentDish(d)"
        >
          <BaseTag :category="d.category" :text="`${d.name} ×${d.count}`" />
        </button>
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
.recent-panel {
  display: flex;
  flex-wrap: wrap;
  align-items: center;
  gap: 8px;
  padding: 10px 12px;
  margin-bottom: 16px;
  background: var(--surface-2);
  border-radius: 8px;
}
.recent-label {
  font-size: 12px;
  color: var(--text-2);
}
.recent-chip {
  border: none;
  background: none;
  padding: 0;
  cursor: pointer;
  transition: transform 0.1s;
}
.recent-chip:hover:not(:disabled) {
  transform: scale(1.05);
}
.recent-chip.added {
  opacity: 0.4;
  cursor: default;
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
</style>
