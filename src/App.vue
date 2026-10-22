
<script setup>
import { ref, computed, watch } from 'vue'

// 当前选中的页面
const activeTab = ref('首页')

// 底部导航
const tabs = [
  { name: '首页', icon: '⌂' },
  { name: '饮食', icon: '♧' },
  { name: '体重', icon: '⚖' },
  { name: '训练', icon: '✦' }
]

// 每日目标
const calorieGoal = 2500
const proteinGoal = 110

// 食物记录列表
// localStorage 中保存数据时使用的固定名称
const STORAGE_KEY = 'gaintrack-foods'

// 页面启动时，尝试读取之前保存的饮食记录
function loadFoods() {
  try {
    const saved = localStorage.getItem(STORAGE_KEY)

    if (saved) {
      const data = JSON.parse(saved)

      // 防止读取到非数组格式的数据
      if (Array.isArray(data)) {
        return data
      }
    }
  } catch (error) {
    console.error('读取饮食记录失败：', error)
  }

  return []
}

// 有历史记录就读取，没有就使用空数组
const foods = ref(loadFoods())
// 常吃食物库：营养值均按每 100g 计算
const foodLibrary = [
  { name: '鸡蛋', kcal: 144, protein: 13.3, note: '可食部分' },
  { name: '香蕉', kcal: 93, protein: 1.4, note: '可食部分' },
  { name: '纯牛奶', kcal: 54, protein: 3.0, note: '参考值' },
  { name: '燕麦片', kcal: 380, protein: 13.5, note: '干重' },
  { name: '米饭', kcal: 116, protein: 2.6, note: '熟重' },
  { name: '鸡胸肉', kcal: 133, protein: 24.6, note: '熟重参考值' },
  { name: '西兰花', kcal: 34, protein: 4.1, note: '参考值' },
  { name: '豆腐', kcal: 84, protein: 8.1, note: '参考值' }
]

// 选择食物后，自动填写营养数据
function selectFood(name) {
  const item = foodLibrary.find(food => food.name === name)

  if (!item) return

  foodForm.value.name = item.name
  foodForm.value.kcalPer100 = item.kcal
  foodForm.value.proteinPer100 = item.protein
}
// 监听饮食记录变化，自动保存到浏览器
watch(
  foods,
  (newFoods) => {
    try {
      localStorage.setItem(
        STORAGE_KEY,
        JSON.stringify(newFoods)
      )
    } catch (error) {
      console.error('保存饮食记录失败：', error)
      alert('保存失败，请检查浏览器存储空间或权限')
    }
  },
  { deep: true }
)
// 新增食物时使用的表单数据
const foodForm = ref({
  meal: '早餐',
  name: '',
  weight: 100,
  kcalPer100: 150,
  proteinPer100: 10
})

// 自动计算当天热量
const totalCalories = computed(() => {
  return foods.value.reduce((total, food) => {
    return total + food.kcalPer100 * food.weight / 100
  }, 0)
})

// 自动计算当天蛋白质
const totalProtein = computed(() => {
  return foods.value.reduce((total, food) => {
    return total + food.proteinPer100 * food.weight / 100
  }, 0)
})

// 计算进度条百分比
const caloriePercent = computed(() => {
  return Math.min(totalCalories.value / calorieGoal * 100, 100)
})

const proteinPercent = computed(() => {
  return Math.min(totalProtein.value / proteinGoal * 100, 100)
})

// 添加食物
function addFood() {
  const form = foodForm.value

  if (!form.name.trim()) {
    alert('请先填写食物名称')
    return
  }

  if (
    !Number.isFinite(form.weight) ||
    !Number.isFinite(form.kcalPer100) ||
    !Number.isFinite(form.proteinPer100) ||
    form.weight <= 0 ||
    form.kcalPer100 < 0 ||
    form.proteinPer100 < 0
  ) {
    alert('请填写有效的重量、热量和蛋白质数据')
    return
  }

  // 将表单数据复制到记录列表
  foods.value.push({
    id: Date.now() + Math.random(),
    meal: form.meal,
    name: form.name.trim(),
    weight: form.weight,
    kcalPer100: form.kcalPer100,
    proteinPer100: form.proteinPer100
  })

  // 清空名称，保留其他输入，方便连续添加
  form.name = ''
}

// 删除一条食物记录
function removeFood(id) {
  foods.value = foods.value.filter(food => food.id !== id)
}
</script>

<template>
  <main class="app">
    <header class="topbar">
      <div>
        <p class="eyebrow">YOUR FITNESS JOURNAL</p>
        <h1>Gain<span>Track</span></h1>
      </div>
      <div class="avatar">G</div>
    </header>

    <section class="greeting">
      <p class="muted">今天也向目标前进一步</p>
      <h2>你的增肌日记 💪</h2>
      <p class="date">{{ new Date().toLocaleDateString('zh-CN') }}</p>
    </section>

    <!-- 首页 -->
    <template v-if="activeTab === '首页'">
      <section class="section">
        <div class="section-heading">
          <h3>今日营养</h3>
          <span class="tag">每日目标</span>
        </div>

        <div class="nutrition-card">
          <div class="nutrition-top">
            <div>
              <p class="muted">已摄入热量</p>
              <strong>{{ Math.round(totalCalories) }}</strong>
              <span class="unit"> / {{ calorieGoal }} kcal</span>
            </div>
            <div class="circle">🔥</div>
          </div>

          <div class="progress">
            <div
              class="progress-fill orange"
              :style="{ width: caloriePercent + '%' }"
            ></div>
          </div>

          <p class="hint">
            还差 {{ Math.max(0, Math.round(calorieGoal - totalCalories)) }} kcal
          </p>

          <div class="divider"></div>

          <div class="nutrition-top">
            <div>
              <p class="muted">蛋白质</p>
              <strong>{{ totalProtein.toFixed(1) }}</strong>
              <span class="unit"> / {{ proteinGoal }} g</span>
            </div>
            <div class="circle green">🥚</div>
          </div>

          <div class="progress">
            <div
              class="progress-fill green-fill"
              :style="{ width: proteinPercent + '%' }"
            ></div>
          </div>

          <p class="hint">
            还差 {{ Math.max(0, (proteinGoal - totalProtein)).toFixed(1) }} g
          </p>

          <button class="primary-button" @click="activeTab = '饮食'">
            ＋ 记录饮食
          </button>
        </div>
      </section>

      <section class="section">
        <div class="section-heading">
          <h3>身体数据</h3>
          <button class="text-button" @click="activeTab = '体重'">
            去记录 →
          </button>
        </div>

        <div class="weight-card">
          <div>
            <p class="muted">体重记录</p>
            <strong class="weight-number">-- <small>kg</small></strong>
            <p class="hint">下一步开发体重打卡</p>
          </div>
          <div class="weight-icon">⚖️</div>
        </div>
      </section>

      <section class="section">
        <div class="section-heading">
          <h3>今日训练</h3>
          <button class="text-button" @click="activeTab = '训练'">
            去记录 →
          </button>
        </div>

        <div class="training-card">
          <div class="training-icon">🏋️</div>
          <div class="training-info">
            <strong>记录今天的训练</strong>
            <p class="muted">下一步开发训练打卡</p>
          </div>
        </div>
      </section>
    </template>

    <!-- 饮食页面 -->
    <template v-else-if="activeTab === '饮食'">
      <section class="section">
        <div class="section-heading">
          <h3>添加食物</h3>
          <span class="tag">饮食记录</span>
        </div>

        <form class="food-form" @submit.prevent="addFood">
          <label>
            用餐时间
            <select v-model="foodForm.meal">
              <option>早餐</option>
              <option>午餐</option>
              <option>晚餐</option>
              <option>加餐</option>
            </select>
          </label>

          <label>
  选择常吃的食物
  <select @change="selectFood($event.target.value)">
    <option value="">选择一种食物</option>
    <option
      v-for="item in foodLibrary"
      :key="item.name"
      :value="item.name"
    >
      {{ item.name }}
    </option>
  </select>
</label>

<label>
  食物名称
  <input
    v-model.trim="foodForm.name"
    placeholder="选择食物后自动填写，也可手动输入"
    required
  />
</label>

          <label>
            食用重量（克）
            <input
              v-model.number="foodForm.weight"
              type="number"
              min="1"
              step="1"
              required
            />
          </label>

          <div class="form-row">
            <label>
              每 100g 热量（kcal）
              <input
                v-model.number="foodForm.kcalPer100"
                type="number"
                min="0"
                step="0.1"
                required
              />
            </label>

            <label>
              每 100g 蛋白质（g）
              <input
                v-model.number="foodForm.proteinPer100"
                type="number"
                min="0"
                step="0.1"
                required
              />
            </label>
          </div>

          <div class="preview">
            <span>本次预计摄入</span>
            <strong>
              {{ Math.round((Number(foodForm.weight) || 0) *
                (Number(foodForm.kcalPer100) || 0) / 100) }} kcal
            </strong>
            <span>
              蛋白质
              {{ ((Number(foodForm.weight) || 0) *
                (Number(foodForm.proteinPer100) || 0) / 100).toFixed(1) }} g
            </span>
          </div>

          <button class="primary-button" type="submit">
            添加到今日记录
          </button>
        </form>
      </section>

      <section class="section">
        <div class="section-heading">
          <h3>今日已记录</h3>
          <span class="tag">{{ foods.length }} 项</span>
        </div>

        <div v-if="foods.length === 0" class="empty-state">
          <div class="empty-icon">🥗</div>
          <strong>还没有饮食记录</strong>
          <p class="muted">添加第一种食物，开始你的增肌记录吧。</p>
        </div>

        <div v-else class="food-list">
          <div
            v-for="food in foods"
            :key="food.id"
            class="food-item"
          >
            <div class="food-info">
              <span class="tag">{{ food.meal }}</span>
              <strong>{{ food.name }}</strong>
              <p class="muted">{{ food.weight }} g</p>
            </div>

            <div class="food-numbers">
              <strong>
                {{ Math.round(food.weight * food.kcalPer100 / 100) }} kcal
              </strong>
              <p class="muted">
                蛋白质
                {{ (food.weight * food.proteinPer100 / 100).toFixed(1) }} g
              </p>
              <button
                class="delete-button"
                type="button"
                @click="removeFood(food.id)"
              >
                删除
              </button>
            </div>
          </div>
        </div>
      </section>
    </template>

    <!-- 暂未开发的模块 -->
    <section v-else class="section">
      <div class="empty-state">
        <div class="empty-icon">
          {{ activeTab === '体重' ? '⚖️' : '🏋️' }}
        </div>
        <strong>{{ activeTab }}模块正在开发中</strong>
        <p class="muted">
          我们会逐步添加记录、历史数据和趋势分析。
        </p>
        <button class="primary-button" @click="activeTab = '首页'">
          返回首页
        </button>
      </div>
    </section>

    <!-- 底部导航 -->
    <nav class="bottom-nav">
      <button
        v-for="tab in tabs"
        :key="tab.name"
        :class="{ selected: activeTab === tab.name }"
        @click="activeTab = tab.name"
      >
        <span class="nav-icon">{{ tab.icon }}</span>
        <span>{{ tab.name }}</span>
      </button>
    </nav>
  </main>
</template>

<style>
:root {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI",
    "PingFang SC", "Microsoft YaHei", sans-serif;
  color: #20251f;
  background: #f5f6f2;
  font-synthesis: none;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
}

button,
input,
select {
  font: inherit;
}

button {
  cursor: pointer;
}

.app {
  width: 100%;
  max-width: 520px;
  min-height: 100vh;
  margin: 0 auto;
  padding: 26px 20px 105px;
}

.topbar,
.section-heading,
.nutrition-top,
.training-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.eyebrow {
  margin: 0 0 5px;
  color: #838a7f;
  font-size: 10px;
  letter-spacing: 2px;
}

h1 {
  margin: 0;
  font-size: 27px;
  letter-spacing: -1px;
}

h1 span {
  color: #65945c;
}

.avatar {
  display: grid;
  width: 42px;
  height: 42px;
  place-items: center;
  border-radius: 15px;
  background: #e2ebdc;
  color: #426b3d;
  font-weight: 700;
}

.greeting {
  margin: 30px 0 25px;
}

.muted,
.hint,
.date {
  color: #858b80;
  font-size: 12px;
}

.greeting .muted {
  margin-bottom: 8px;
}

h2 {
  margin: 0;
  font-size: 23px;
}

.date {
  margin-top: 10px;
}

.section {
  margin-top: 25px;
}

.section-heading {
  margin-bottom: 13px;
}

h3 {
  margin: 0;
  font-size: 16px;
}

.tag {
  padding: 6px 10px;
  border-radius: 20px;
  background: #e8eee3;
  color: #5d8053;
  font-size: 11px;
}

.nutrition-card,
.weight-card,
.training-card,
.food-form,
.empty-state {
  padding: 19px;
  border: 1px solid #edf0e9;
  border-radius: 20px;
  background: #fff;
}

.nutrition-top {
  gap: 12px;
}

.nutrition-top strong {
  font-size: 27px;
}

.unit {
  color: #8b9187;
  font-size: 12px;
}

.circle,
.weight-icon,
.training-icon {
  display: grid;
  width: 42px;
  height: 42px;
  flex-shrink: 0;
  place-items: center;
  border-radius: 14px;
  background: #fff0df;
  font-size: 20px;
}

.green {
  background: #e7f2e2;
}

.progress {
  height: 7px;
  margin-top: 13px;
  overflow: hidden;
  border-radius: 10px;
  background: #eef0eb;
}

.progress-fill {
  height: 100%;
  border-radius: inherit;
  transition: width 0.3s ease;
}

.orange {
  background: #e9a45e;
}

.green-fill {
  background: #79a96b;
}

.hint {
  margin: 9px 0 0;
}

.divider {
  height: 1px;
  margin: 20px 0;
  background: #eef0eb;
}

.text-button {
  padding: 4px 0;
  border: 0;
  background: transparent;
  color: #688a5d;
  font-size: 12px;
}

.weight-card {
  display: flex;
  align-items: center;
  justify-content: space-between;
}

.weight-number {
  display: block;
  margin-top: 8px;
  font-size: 30px;
}

.weight-number small {
  font-size: 14px;
  font-weight: 400;
}

.weight-icon {
  width: 52px;
  height: 52px;
  background: #edf2e8;
}

.training-card {
  gap: 12px;
}

.training-icon {
  background: #f0eafa;
}

.training-info {
  flex: 1;
}

.training-info strong {
  font-size: 13px;
}

.training-info p {
  margin-bottom: 0;
  line-height: 1.6;
}

.bottom-nav {
  position: fixed;
  right: 0;
  bottom: 0;
  left: 0;
  z-index: 10;
  display: flex;
  justify-content: center;
  padding: 10px 12px calc(10px + env(safe-area-inset-bottom));
  border-top: 1px solid #e9ece5;
  background: rgba(255, 255, 255, 0.96);
}

.bottom-nav button {
  display: flex;
  width: 25%;
  max-width: 120px;
  flex-direction: column;
  align-items: center;
  gap: 4px;
  padding: 5px 0;
  border: 0;
  background: transparent;
  color: #93998f;
  font-size: 11px;
}

.bottom-nav button.selected {
  color: #59854e;
  font-weight: 600;
}

.nav-icon {
  font-size: 20px;
}

.primary-button {
  width: 100%;
  margin-top: 18px;
  padding: 13px;
  border: 0;
  border-radius: 13px;
  background: #688f5c;
  color: white;
  font-size: 14px;
  font-weight: 600;
}

.primary-button:active {
  opacity: 0.85;
}

.food-form {
  display: flex;
  flex-direction: column;
  gap: 15px;
}

.food-form label {
  display: flex;
  flex: 1;
  flex-direction: column;
  gap: 8px;
  color: #535b50;
  font-size: 12px;
}

.food-form input,
.food-form select {
  width: 100%;
  min-width: 0;
  padding: 12px;
  border: 1px solid #e5e9e0;
  border-radius: 11px;
  outline: none;
  background: #fafbf8;
  color: #20251f;
  font-size: 14px;
}

.food-form input:focus,
.food-form select:focus {
  border-color: #82a876;
}

.form-row {
  display: flex;
  gap: 12px;
}

.preview {
  display: flex;
  flex-direction: column;
  gap: 7px;
  padding: 14px;
  border-radius: 13px;
  background: #f1f5ed;
  color: #65745e;
  font-size: 12px;
}

.preview strong {
  color: #456c3d;
  font-size: 23px;
}

.food-list {
  display: flex;
  flex-direction: column;
  gap: 10px;
}

.food-item {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 10px;
  padding: 15px;
  border: 1px solid #edf0e9;
  border-radius: 16px;
  background: #fff;
}

.food-info,
.food-numbers {
  display: flex;
  flex-direction: column;
  align-items: flex-start;
  gap: 7px;
}

.food-info {
  min-width: 0;
}

.food-info strong {
  overflow-wrap: anywhere;
}

.food-info p,
.food-numbers p {
  margin: 0;
}

.food-numbers {
  align-items: flex-end;
  flex-shrink: 0;
}

.food-numbers strong {
  font-size: 14px;
}

.delete-button {
  padding: 4px 0;
  border: 0;
  background: transparent;
  color: #b55e52;
  font-size: 12px;
}

.empty-state {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
}

.empty-icon {
  margin-bottom: 12px;
  font-size: 32px;
}

.empty-state strong {
  font-size: 15px;
}

.empty-state p {
  line-height: 1.7;
}

@media (min-width: 600px) {
  .app {
    margin-top: 20px;
    margin-bottom: 20px;
    border-radius: 24px;
    background: #f5f6f2;
  }
}
</style>
