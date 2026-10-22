
<script setup>
import { ref, computed, watch } from 'vue'

const STORAGE_KEY = 'gaintrack-workouts-v1'

function loadWorkouts() {
  try {
    return JSON.parse(localStorage.getItem(STORAGE_KEY) || '[]')
  } catch {
    return []
  }
}

function createId() {
  return `${Date.now()}-${Math.random().toString(36).slice(2, 9)}`
}

function getToday() {
  const date = new Date()
  const year = date.getFullYear()
  const month = String(date.getMonth() + 1).padStart(2, '0')
  const day = String(date.getDate()).padStart(2, '0')
  return `${year}-${month}-${day}`
}

const workouts = ref(loadWorkouts())
const activeTab = ref('home')
const editingWorkoutId = ref(null)




const workoutForm = ref({
  date: getToday(),
  muscle: '胸部',
  note: ''
})

const exerciseForm = ref({
  name: '',
  initialSets: 3
})

const exercises = ref([])
watch(
  [workoutForm, exerciseForm, exercises, editingWorkoutId],
  () => {
    if (isResettingWorkout) return
    saveWorkoutDraft()
  },
  { deep: true }
)
const DRAFT_KEY = 'gaintrack_workout_draft'
let isResettingWorkout = false
// 保存当前训练草稿
function saveWorkoutDraft() {
  // 没有正在编辑的动作时，也保存日期、部位等表单信息
  const draft = {
    editingWorkoutId: editingWorkoutId.value,
    workoutForm: JSON.parse(JSON.stringify(workoutForm.value)),
    exerciseForm: JSON.parse(JSON.stringify(exerciseForm.value)),
    exercises: JSON.parse(JSON.stringify(exercises.value))
  }

  localStorage.setItem(DRAFT_KEY, JSON.stringify(draft))
}

// 恢复上次未完成的训练草稿
function restoreWorkoutDraft() {
  const savedDraft = localStorage.getItem(DRAFT_KEY)

  if (!savedDraft) return false

  try {
    const draft = JSON.parse(savedDraft)

    if (!draft || !draft.workoutForm || !Array.isArray(draft.exercises)) {
      return false
    }

    editingWorkoutId.value = draft.editingWorkoutId ?? null
    workoutForm.value = draft.workoutForm
    exerciseForm.value = draft.exerciseForm ?? {
      name: '',
      initialSets: 3
    }
    exercises.value = draft.exercises

    return true
  } catch (error) {
    console.error('恢复训练草稿失败：', error)
    localStorage.removeItem(DRAFT_KEY)
    return false
  }
}

// 清除训练草稿
function clearWorkoutDraft() {
  localStorage.removeItem(DRAFT_KEY)
}

const muscleOptions = ['胸部', '背部', '肩部', '手臂', '腿部', '全身']

// 本周一至下周一，用于统计本周训练
const weekWorkouts = computed(() => {
  const now = new Date()
  const monday = new Date(now)
  const day = (now.getDay() + 6) % 7

  monday.setDate(now.getDate() - day)
  monday.setHours(0, 0, 0, 0)

  const nextMonday = new Date(monday)
  nextMonday.setDate(monday.getDate() + 7)

  return workouts.value.filter((workout) => {
    const date = new Date(`${workout.date}T12:00:00`)
    return date >= monday && date < nextMonday
  })
})

const totalSetsThisWeek = computed(() =>
  weekWorkouts.value.reduce(
    (total, workout) =>
      total +
      workout.exercises.reduce(
        (sum, exercise) => sum + exercise.sets.length,
        0
      ),
    0
  )
)

const latestWorkout = computed(() =>
  [...workouts.value].sort((a, b) =>
    b.date.localeCompare(a.date)
  )[0] || null
)

const sortedWorkouts = computed(() =>
  [...workouts.value].sort((a, b) =>
    b.date.localeCompare(a.date)
  )
)

const exerciseNames = computed(() => {
  const names = new Set()

  // 收集已经保存的历史动作
  for (const workout of workouts.value) {
    for (const exercise of workout.exercises) {
      if (exercise.name.trim()) {
        names.add(exercise.name.trim())
      }
    }
  }

  // 也收集本次训练中已经添加的动作
  for (const exercise of exercises.value) {
    if (exercise.name.trim()) {
      names.add(exercise.name.trim())
    }
  }

  return [...names].sort((a, b) => a.localeCompare(b, 'zh-CN'))
})


// 查找当前动作在本次训练日期之前的最近一次记录

function getPreviousExercise(name) {
  const normalizedName = name.trim().toLowerCase()
  if (!normalizedName) return null

  const matches = []

  workouts.value.forEach((workout, workoutIndex) => {
    // 编辑记录时，不能把正在编辑的记录当成上次训练
    if (workout.id === editingWorkoutId.value) return

    // 允许查询同一天更早保存的训练
    if (workout.date > workoutForm.value.date) return

    workout.exercises.forEach((exercise, exerciseIndex) => {
      if (exercise.name.trim().toLowerCase() === normalizedName) {
        matches.push({
          ...exercise,
          date: workout.date,
          muscle: workout.muscle,
          workoutIndex,
          exerciseIndex
        })
      }
    })
  })

  // 日期较新的优先；同一天则优先显示后保存的训练
  matches.sort((a, b) =>
    b.date.localeCompare(a.date) ||
    b.workoutIndex - a.workoutIndex ||
    b.exerciseIndex - a.exerciseIndex
  )

  return matches[0] || null
}



function addExercise() {
  const name = exerciseForm.value.name.trim()
  const count = Number(exerciseForm.value.initialSets)

  if (!name) {
    alert('请先填写动作名称')
    return
  }

  if (!Number.isInteger(count) || count < 1 || count > 20) {
    alert('组数请填写 1～20 之间的整数')
    return
  }

  exercises.value.push({
    id: createId(),
    name,
    sets: Array.from({ length: count }, () => ({
      id: createId(),
      weight: '',
      reps: '',
      weightLinked: true,
      repsLinked: true
    }))
  })

  exerciseForm.value.name = ''
  exerciseForm.value.initialSets = 3
}


function addSet(exercise) {
  const previous = exercise.sets.at(-1)

  exercise.sets.push({
    id: createId(),
    weight: previous?.weight ?? '',
    reps: previous?.reps ?? '',
    weightLinked: false,
    repsLinked: false
  })
}

function removeSet(exercise, setIndex) {
  if (exercise.sets.length === 1) {
    alert('每个动作至少保留一组')
    return
  }

  exercise.sets.splice(setIndex, 1)
}


function updateSetWeight(exercise, setIndex, value) {
  const set = exercise.sets[setIndex]
  set.weight = value

  if (setIndex !== 0) {
    set.weightLinked = false
  }
}

function updateSetReps(exercise, setIndex, value) {
  const set = exercise.sets[setIndex]
  set.reps = value

  if (setIndex !== 0) {
    set.repsLinked = false
  }
}


// 同步第一组的值到仍处于跟随状态的组
function syncFirstSet(exercise) {
  const first = exercise.sets[0]
  if (!first) return

  exercise.sets.slice(1).forEach((set) => {
    if (set.weightLinked) set.weight = first.weight
    if (set.repsLinked) set.reps = first.reps
  })
}

function removeExercise(index) {
  exercises.value.splice(index, 1)
}



function startNewWorkout(forceNew = false) {
  // 非强制新建时，优先恢复未完成的草稿
  if (!forceNew) {
    const restored = restoreWorkoutDraft()

    if (restored) {
      activeTab.value = 'train'
      return
    }
  }

  editingWorkoutId.value = null
  workoutForm.value = {
    date: getToday(),
    muscle: '胸部',
    note: ''
  }
  exercises.value = []
  exerciseForm.value = {
    name: '',
    initialSets: 3
  }

  activeTab.value = 'train'
}

function openWorkout() {
  // 如果当前已经在训练页面，不重置表单
  if (activeTab.value === 'train') return

  // 如果当前不是训练页面，尝试恢复未完成的草稿
  const restored = restoreWorkoutDraft()

  if (!restored) {
    startNewWorkout(true)
    return
  }

  activeTab.value = 'train'
}


function editWorkout(workout) {
  editingWorkoutId.value = workout.id
  workoutForm.value = {
    date: workout.date,
    muscle: workout.muscle,
    note: workout.note || ''
  }
  exercises.value = JSON.parse(JSON.stringify(workout.exercises))
  exerciseForm.value = {
    name: '',
    initialSets: 3
  }
  activeTab.value = 'train'
}

function saveWorkout() {
  if (!workoutForm.value.date) {
    alert('请选择训练日期')
    return
  }

  if (exercises.value.length === 0) {
    alert('请至少添加一个训练动作')
    return
  }

  for (const exercise of exercises.value) {
    if (!exercise.name.trim() || exercise.sets.length === 0) {
      alert('请检查动作名称和训练组数')
      return
    }

    for (const set of exercise.sets) {
      if (
        set.weight === '' ||
        set.reps === '' ||
        !Number.isFinite(Number(set.weight)) ||
        !Number.isFinite(Number(set.reps)) ||
        Number(set.weight) < 0 ||
        Number(set.reps) < 1 ||
        !Number.isInteger(Number(set.reps))
      ) {
        alert(`请检查「${exercise.name}」的重量和次数。次数必须是大于 0 的整数。`)
        return
      }
    }
  }

  const workout = {
    id: editingWorkoutId.value || createId(),
    date: workoutForm.value.date,
    muscle: workoutForm.value.muscle,
    note: workoutForm.value.note.trim(),
    exercises: JSON.parse(JSON.stringify(exercises.value))
  }

  if (editingWorkoutId.value) {
    const index = workouts.value.findIndex(
      (item) => item.id === editingWorkoutId.value
    )

    if (index !== -1) {
      workouts.value[index] = workout
    }
  } else {
    workouts.value.push(workout)
  }

clearWorkoutDraft()
startNewWorkout(true)
activeTab.value = 'history'
alert('训练记录已保存！')
}

function deleteWorkout(id) {
  if (!confirm('确定删除这条训练记录吗？')) return

  workouts.value = workouts.value.filter((item) => item.id !== id)
}

function formatDate(date) {
  if (!date) return ''
  const [year, month, day] = date.split('-')
  return `${year}/${month}/${day}`
}

function totalSets(workout) {
  return workout.exercises.reduce(
    (sum, exercise) => sum + exercise.sets.length,
    0
  )
}
</script>

<template>
  <div class="app">
    <header class="topbar">
      <div>
        <p class="eyebrow">YOUR FITNESS JOURNAL</p>
        <h1>Gain<span>Track</span></h1>
      </div>
      <div class="avatar">GT</div>
    </header>

    <main>
      <!-- 首页 -->
      <section v-if="activeTab === 'home'">
        <div class="welcome">
          <p class="muted">今天也向目标靠近一点</p>
          <h2>练得有记录，进步看得见。</h2>
          <button class="primary-button" @click="startNewWorkout">
            ＋ 记录训练
          </button>
        </div>

        <div class="stats-grid">
          <article class="stat-card">
            <span>本周训练</span>
            <strong>{{ weekWorkouts.length }}<small> 次</small></strong>
          </article>
          <article class="stat-card">
            <span>本周总组数</span>
            <strong>{{ totalSetsThisWeek }}<small> 组</small></strong>
          </article>
        </div>

        <div class="section-heading">
          <h2>最近一次训练</h2>
          <button class="text-button" @click="activeTab = 'history'">
            查看全部
          </button>
        </div>

        <article v-if="latestWorkout" class="workout-card">
          <div class="card-top">
            <div>
              <span class="tag">{{ latestWorkout.muscle }}</span>
              <h3>{{ formatDate(latestWorkout.date) }}</h3>
            </div>
            <span class="muted">{{ totalSets(latestWorkout) }} 组</span>
          </div>

          <div
            v-for="exercise in latestWorkout.exercises"
            :key="exercise.id"
            class="exercise-summary"
          >
            <strong>{{ exercise.name }}</strong>
            <span>
              {{ exercise.sets.length }} 组 ·
              {{ exercise.sets.map(s => `${s.weight}kg × ${s.reps}`).join(' / ') }}
            </span>
          </div>

          <p v-if="latestWorkout.note" class="note">
            {{ latestWorkout.note }}
          </p>

          <button
            class="secondary-button full-button"
            @click="editWorkout(latestWorkout)"
          >
            编辑这次训练
          </button>
        </article>

        <div v-else class="empty-state">
          <div class="empty-icon">🏋️</div>
          <h3>还没有训练记录</h3>
          <p>完成第一次训练记录，就能开始追踪自己的进步。</p>
        </div>
      </section>

      <!-- 训练记录 -->
      <section v-else-if="activeTab === 'train'">
        <div class="page-heading">
          <p class="eyebrow">WORKOUT LOG</p>
          <h2>{{ editingWorkoutId ? '编辑训练' : '记录训练' }}</h2>
          <p class="muted">每一组都认真记录，每一次都能复盘。</p>
        </div>

        <div class="form-card">
          <label class="field-label" for="workout-date">训练日期</label>
          <input id="workout-date" v-model="workoutForm.date" type="date" />

          <label class="field-label" for="muscle">训练部位</label>
          <select id="muscle" v-model="workoutForm.muscle">
            <option v-for="muscle in muscleOptions" :key="muscle" :value="muscle">
              {{ muscle }}
            </option>
          </select>

          <label class="field-label" for="workout-note">训练备注</label>
          <textarea
            id="workout-note"
            v-model="workoutForm.note"
            rows="2"
            placeholder="例如：状态不错，最后一组比较吃力"
          />
        </div>

        <div class="section-heading">
          <h2>训练动作</h2>
          <span class="muted">{{ exercises.length }} 个动作</span>
        </div>

        <article
          v-for="(exercise, index) in exercises"
          :key="exercise.id"
          class="exercise-card"
        >
          <div class="exercise-card-heading">
            <div>
              <span class="exercise-number">动作 {{ index + 1 }}</span>
              <h3>{{ exercise.name }}</h3>
            </div>
            <button
              class="icon-button"
              aria-label="删除动作"
              @click="removeExercise(index)"
            >
              ×
            </button>
          </div>

          <div v-if="getPreviousExercise(exercise.name)" class="previous-record">
            <span>上次训练 · {{ formatDate(getPreviousExercise(exercise.name).date) }}</span>
            <strong>
              {{ getPreviousExercise(exercise.name).sets.length }} 组：
              {{ getPreviousExercise(exercise.name).sets.map(s => `${s.weight}kg × ${s.reps}`).join(' / ') }}
            </strong>
          </div>

          <div v-else class="previous-record no-record">
            暂无更早的同名动作记录
          </div>

          <div class="sets-header">
            <span>组数</span>
            <span>重量（kg）</span>
            <span>次数</span>
            <span></span>
          </div>

          <div
            v-for="(set, setIndex) in exercise.sets"
            :key="set.id"
            class="set-row"
          >
            <span class="set-index">{{ setIndex + 1 }}</span>
            
<input
  :value="set.weight"
  type="number"
  min="0"
  step="0.5"
  placeholder="重量"
  :aria-label="`第${setIndex + 1}组重量`"
  @input="updateSetWeight(exercise, setIndex, $event.target.value); setIndex === 0 && syncFirstSet(exercise)"
/>

<input
  :value="set.reps"
  type="number"
  min="1"
  step="1"
  placeholder="次数"
  :aria-label="`第${setIndex + 1}组次数`"
  @input="updateSetReps(exercise, setIndex, $event.target.value); setIndex === 0 && syncFirstSet(exercise)"
/>

            <button
              class="remove-set-button"
              :aria-label="`删除第${setIndex + 1}组`"
              @click="removeSet(exercise, setIndex)"
            >
              −
            </button>
          </div>

          <button class="add-set-button" @click="addSet(exercise)">
            ＋ 添加一组
          </button>
        </article>

        <div class="add-exercise-card">
          <h3>添加动作</h3>
          
<label class="field-label" for="exercise-name">动作名称</label>

<input
  id="exercise-name"
  v-model="exerciseForm.name"
  list="exercise-options"
  placeholder="选择已有动作，或输入新动作"
  @keydown.enter.prevent="addExercise"
/>

<datalist id="exercise-options">
  <option
    v-for="name in exerciseNames"
    :key="name"
    :value="name"
  />
</datalist>

          <label class="field-label" for="initial-sets">初始组数</label>
          <select id="initial-sets" v-model.number="exerciseForm.initialSets">
            <option v-for="count in [1, 2, 3, 4, 5, 6, 8, 10]" :key="count" :value="count">
              {{ count }} 组
            </option>
          </select>

          <button class="secondary-button full-button" @click="addExercise">
            ＋ 添加到本次训练
          </button>
        </div>

        <button class="primary-button full-button save-button" @click="saveWorkout">
          {{ editingWorkoutId ? '保存修改' : '保存训练记录' }}
        </button>
        <button class="text-button cancel-button" @click="activeTab = 'home'">
          返回首页
        </button>
      </section>

      <!-- 历史记录 -->
      <section v-else>
        <div class="page-heading">
          <p class="eyebrow">TRAINING HISTORY</p>
          <h2>训练历史</h2>
          <p class="muted">回顾训练，比较表现，慢慢进步。</p>
        </div>

        <div v-if="sortedWorkouts.length" class="history-list">
          <article
            v-for="workout in sortedWorkouts"
            :key="workout.id"
            class="workout-card"
          >
            <div class="card-top">
              <div>
                <span class="tag">{{ workout.muscle }}</span>
                <h3>{{ formatDate(workout.date) }}</h3>
              </div>
              <span class="muted">{{ totalSets(workout) }} 组</span>
            </div>

            <div
              v-for="exercise in workout.exercises"
              :key="exercise.id"
              class="exercise-summary"
            >
              <strong>{{ exercise.name }}</strong>
              <span>
                {{ exercise.sets.map(s => `${s.weight}kg × ${s.reps}`).join(' / ') }}
              </span>
            </div>

            <p v-if="workout.note" class="note">{{ workout.note }}</p>

            <div class="history-actions">
              <button class="secondary-button" @click="editWorkout(workout)">
                编辑
              </button>
              <button class="danger-button" @click="deleteWorkout(workout.id)">
                删除
              </button>
            </div>
          </article>
        </div>

        <div v-else class="empty-state">
          <div class="empty-icon">📋</div>
          <h3>这里还空空的</h3>
          <p>保存训练后，所有历史记录都会显示在这里。</p>
          <button class="primary-button" @click="startNewWorkout">
            开始记录
          </button>
        </div>
      </section>
    </main>

    <nav class="bottom-nav">
      <button :class="{ active: activeTab === 'home' }" @click="activeTab = 'home'">
        <span>⌂</span>首页
      </button>
      <button :class="{ active: activeTab === 'train' }" @click="openWorkout">
        <span>＋</span>训练
      </button>
      <button :class="{ active: activeTab === 'history' }" @click="activeTab = 'history'">
        <span>▤</span>历史
      </button>
    </nav>
  </div>
</template>

<style>
:root {
  font-family: Inter, -apple-system, BlinkMacSystemFont, "Segoe UI", sans-serif;
  color: #f4f5f7;
  background: #101214;
  font-synthesis: none;
  text-rendering: optimizeLegibility;
  -webkit-font-smoothing: antialiased;
  --panel: #1b1e22;
  --panel-light: #24282d;
  --muted: #9299a3;
  --accent: #c4f36b;
  --border: #30353b;
}

* {
  box-sizing: border-box;
}

body {
  margin: 0;
  min-width: 320px;
  min-height: 100vh;
}

button, input, select, textarea {
  font: inherit;
}

button {
  cursor: pointer;
}

.app {
  width: 100%;
  max-width: 620px;
  min-height: 100vh;
  margin: 0 auto;
  padding: 24px 18px 110px;
}

.topbar, .card-top, .section-heading, .exercise-card-heading {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 12px;
}

.topbar {
  margin-bottom: 30px;
}

.eyebrow {
  margin: 0 0 7px;
  color: var(--accent);
  font-size: 10px;
  font-weight: 800;
  letter-spacing: 2px;
}

h1 {
  margin: 0;
  font-size: 29px;
  letter-spacing: -1.2px;
}

h1 span {
  color: var(--accent);
}

.avatar {
  display: grid;
  place-items: center;
  width: 42px;
  height: 42px;
  border: 1px solid #424a3b;
  border-radius: 14px;
  color: var(--accent);
  background: #242a20;
  font-size: 13px;
  font-weight: 800;
}

h2, h3, p {
  overflow-wrap: anywhere;
}

.welcome {
  padding: 24px 20px;
  border: 1px solid #35402a;
  border-radius: 22px;
  background: linear-gradient(135deg, #242c20, #1b1e22 75%);
}

.welcome h2 {
  max-width: 340px;
  margin: 10px 0 22px;
  font-size: 25px;
  line-height: 1.35;
}

.muted {
  color: var(--muted);
  font-size: 12px;
}

.stats-grid {
  display: grid;
  grid-template-columns: 1fr 1fr;
  gap: 12px;
  margin: 16px 0 30px;
}

.stat-card, .workout-card, .form-card, .exercise-card, .add-exercise-card {
  border: 1px solid var(--border);
  border-radius: 18px;
  background: var(--panel);
}

.stat-card {
  padding: 17px;
}

.stat-card span {
  display: block;
  color: var(--muted);
  font-size: 12px;
}

.stat-card strong {
  display: block;
  margin-top: 12px;
  font-size: 29px;
}

.stat-card small {
  color: var(--muted);
  font-size: 12px;
  font-weight: 500;
}

.section-heading {
  margin: 25px 0 13px;
}

.section-heading h2, .page-heading h2 {
  margin: 0;
  font-size: 19px;
}

.text-button {
  padding: 8px 0;
  border: 0;
  color: var(--accent);
  background: transparent;
  font-size: 13px;
}

.primary-button, .secondary-button, .danger-button {
  min-height: 42px;
  padding: 11px 15px;
  border: 0;
  border-radius: 12px;
  font-size: 13px;
  font-weight: 750;
}

.primary-button {
  color: #171b11;
  background: var(--accent);
}

.secondary-button {
  color: #f0f1f3;
  background: #30353b;
}

.danger-button {
  color: #ff9999;
  background: #3a2427;
}

.full-button {
  width: 100%;
}

.workout-card {
  padding: 17px;
  margin-bottom: 13px;
}

.card-top {
  align-items: flex-start;
  margin-bottom: 16px;
}

.card-top h3 {
  margin: 10px 0 0;
  font-size: 17px;
}

.tag {
  display: inline-block;
  padding: 5px 9px;
  border-radius: 7px;
  color: var(--accent);
  background: #2b3522;
  font-size: 11px;
}

.exercise-summary {
  display: flex;
  flex-direction: column;
  gap: 5px;
  padding: 12px 0;
  border-top: 1px solid var(--border);
}

.exercise-summary strong {
  font-size: 13px;
}

.exercise-summary span {
  color: var(--muted);
  font-size: 12px;
  line-height: 1.6;
  overflow-wrap: anywhere;
}

.note {
  margin: 12px 0;
  color: #bdc2c9;
  font-size: 12px;
  line-height: 1.7;
}

.empty-state {
  padding: 40px 22px;
  border: 1px dashed #41464d;
  border-radius: 18px;
  text-align: center;
}

.empty-icon {
  margin-bottom: 14px;
  font-size: 32px;
}

.empty-state h3 {
  margin: 0 0 10px;
}

.empty-state p {
  margin: 0 0 20px;
  color: var(--muted);
  font-size: 13px;
  line-height: 1.7;
}

.page-heading {
  margin-bottom: 24px;
}

.page-heading h2 {
  margin-bottom: 8px;
  font-size: 27px;
}

.page-heading > p:last-child {
  line-height: 1.6;
}

.form-card, .add-exercise-card {
  padding: 18px;
  margin-bottom: 22px;
}

.field-label {
  display: block;
  margin: 15px 0 8px;
  color: #b6bcc4;
  font-size: 12px;
  font-weight: 650;
}

.field-label:first-child {
  margin-top: 0;
}

input, select, textarea {
  width: 100%;
  min-width: 0;
  padding: 12px;
  border: 1px solid #3a4047;
  border-radius: 10px;
  outline: none;
  color: #f4f5f7;
  background: #141619;
  font-size: 14px;
}

input:focus, select:focus, textarea:focus {
  border-color: var(--accent);
}

textarea {
  resize: vertical;
}

.exercise-card {
  padding: 16px;
  margin-bottom: 13px;
}

.exercise-card-heading h3 {
  margin: 5px 0 0;
  font-size: 16px;
}

.exercise-number {
  color: var(--muted);
  font-size: 11px;
}

.icon-button {
  width: 32px;
  height: 32px;
  border: 1px solid var(--border);
  border-radius: 10px;
  color: #ff9999;
  background: #2a2224;
  font-size: 20px;
}

.previous-record {
  display: flex;
  flex-direction: column;
  gap: 6px;
  margin: 14px 0;
  padding: 12px;
  border: 1px solid #3c4a2d;
  border-radius: 10px;
  background: #232a1e;
}

.previous-record span {
  color: var(--accent);
  font-size: 11px;
}

.previous-record strong {
  font-size: 12px;
  line-height: 1.7;
  overflow-wrap: anywhere;
}

.previous-record.no-record {
  border-color: var(--border);
  color: var(--muted);
  background: #202327;
  font-size: 12px;
}

.sets-header, .set-row {
  display: grid;
  grid-template-columns: 36px minmax(0, 1fr) minmax(0, 1fr) 30px;
  align-items: center;
  gap: 7px;
}

.sets-header {
  margin: 18px 0 8px;
  color: var(--muted);
  font-size: 10px;
  text-align: center;
}

.set-row {
  margin-bottom: 8px;
}

.set-index {
  color: var(--muted);
  font-size: 12px;
  text-align: center;
}

.set-row input {
  padding: 10px 4px;
  text-align: center;
  font-size: 13px;
}

.remove-set-button {
  width: 28px;
  height: 34px;
  border: 0;
  border-radius: 8px;
  color: #ff9999;
  background: #f5f1f1ff;
  font-size: 19px;
}

.add-set-button {
  width: 100%;
  padding: 11px;
  margin-top: 6px;
  border: 1px dashed #48503d;
  border-radius: 10px;
  color: var(--accent);
  background: transparent;
  font-size: 12px;
}

.add-exercise-card h3 {
  margin: 0 0 5px;
  font-size: 16px;
}

.add-exercise-card .secondary-button {
  margin-top: 18px;
}

.save-button {
  margin-top: 10px;
}

.cancel-button {
  display: block;
  margin: 8px auto 0;
}

.history-list {
  display: flex;
  flex-direction: column;
  gap: 12px;
}

.history-actions {
  display: flex;
  gap: 8px;
  margin-top: 14px;
}

.history-actions button {
  flex: 1;
}

.bottom-nav {
  position: fixed;
  z-index: 10;
  right: 0;
  bottom: 0;
  left: 0;
  display: flex;
  justify-content: center;
  gap: 0;
  padding: 10px 10px calc(10px + env(safe-area-inset-bottom));
  border-top: 1px solid #2c3035;
  background: rgba(20, 22, 25, 0.96);
  backdrop-filter: blur(18px);
}

.bottom-nav button {
  display: flex;
  flex: 1;
  max-width: 200px;
  flex-direction: column;
  align-items: center;
  gap: 5px;
  border: 0;
  color: #858c95;
  background: transparent;
  font-size: 10px;
}

.bottom-nav button span {
  font-size: 22px;
}

.bottom-nav button.active {
  color: var(--accent);
}

@media (min-width: 620px) {
  .app {
    padding-top: 32px;
  }
}

/* ===== GainTrack 浅色主题 ===== */
:root {
  color: #202522;
  background: #f5f7f4;
  --panel: #ffffff;
  --panel-light: #f1f4ef;
  --muted: #010902ff;
  --accent: #58852c;
  --border: #e1e6df;
}

body {
  color: #202522;
  background: #f5f7f4;
}

.welcome {
  border-color: #dce8d0;
  background: linear-gradient(135deg, #edf5e5, #ffffff);
}

.avatar {
  border-color: #dce8d0;
  color: #58852c;
  background: #edf5e5;
}

.stat-card,
.workout-card,
.form-card,
.exercise-card,
.add-exercise-card {
  border-color: #e1e6df;
  background: #ffffff;
}

.tag {
  color: #4f782a;
  background: #edf5e5;
}

.primary-button {
  color: #ffffff;
  background: #58852c;
}

.secondary-button {
  color: #343a35;
  background: #edf0eb;
}

input,
select,
textarea {
  color: #202522;
  border-color: #dce2da;
  background: #ffffff;
}

input:focus,
select:focus,
textarea:focus {
  border-color: #58852c;
}

.previous-record {
  border-color: #dce8d0;
  background: #f1f7eb;
}

.previous-record span {
  color: #4f782a;
}

.previous-record.no-record {
  border-color: #e1e6df;
  color: #747c75;
  background: #f6f7f5;
}

.add-set-button {
  border-color: #cbd9bf;
  color: #4f782a;
}

.icon-button,
.remove-set-button {
  color: #b83d3d;
  border-color: #f0dada;
  background: #fff4f4;
}

.danger-button {
  color: #b83d3d;
  background: #fff0f0;
}

.empty-state {
  border-color: #dce2da;
}

.bottom-nav {
  border-top-color: #e1e6df;
  background: rgba(255, 255, 255, 0.96);
}

.bottom-nav button {
  color: #747c75;
}

.bottom-nav button.active,
.text-button,
.cancel-button {
  color: #58852c;
}

.exercise-summary {
  border-top-color: #e9ede7;
}

</style>
