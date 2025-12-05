<template>
  <div class="layout">
    <div v-for="y in years" :key="y">
      <h1>{{ y }} 年日历</h1>
      <div class="calendar-year" :style="{ '--cell-size': yearCellSize(y) + 'px' }">
        <div class="month" v-for="m in 12" :key="m">
          <div class="month-header">{{ monthNames[m - 1] }}</div>
          <div class="weekdays">
            <div class="weekday" v-for="w in weekdays" :key="w">{{ w }}</div>
          </div>
          <div class="days">
            <div class="day placeholder" v-for="n in firstWeekday(y, m)" :key="`p-${m}-${n}`"></div>
            <div
              v-for="d in daysInMonth(y, m)"
              :key="`d-${m}-${d}`"
              class="day"
              :class="dayClass(y, m, d) + (isSelected(y, m, d) ? ' selected' : '')"
              :data-date="fmt(y, m, d)"
              @click="onSelectDay(y, m, d)"
            >
              <div class="date">{{ d }}</div>
              <div class="marks">
                <div class="mark" v-if="holidayLabel(y, m, d)">{{ holidayLabel(y, m, d) }}</div>
                <div class="workday" v-if="isAdjustWorkday(y, m, d)">调休上班</div>
                <div class="events" v-if="eventsCount(y, m, d) > 0">
                  <div class="event" v-for="(ev, idx) in eventsList(y, m, d)" :key="ev.key" :class="{ red: ev.red, done: isDone(ev.key) }">
                    <span class="event-number">{{ idx + 1 }}.</span>
                    <span class="event-text">{{ ev.text }}</span>
                    <input type="checkbox" class="event-check" :checked="isDone(ev.key)" @change="toggleDone(ev.key, $event.target.checked)" />
                  </div>
                </div>
              </div>
            </div>
          </div>
        </div>
      </div>
    </div>

    <div class="side-panel" :style="{ '--panel-x': state.panel.x + 'px', '--panel-y': state.panel.y + 'px', '--panel-scale': state.panel.scale }">
      <div class="panel-header" @mousedown="startDragPanel">
        <div class="title">计划设置</div>
        <div class="actions">
          <button class="btn" @click.stop="zoomOut">-</button>
          <button class="btn" @click.stop="zoomIn">+</button>
          <button class="btn" @click.stop="resetPanel">重置</button>
          <button class="btn" @click.stop="scrollToToday">今天</button>
          <button class="btn" @click.stop="state.panel.minimized = !state.panel.minimized">{{ state.panel.minimized ? '展开' : '收起' }}</button>
        </div>
      </div>
      <div v-if="!state.panel.minimized">
        <div class="divider"></div>
        <div v-if="supa">
          <div v-if="!state.auth.user">
            <div class="hint">注册或登录</div>
            <div class="row"><label>邮箱</label><input type="email" v-model="state.auth.email" class="text" /></div>
            <div class="row"><label>用户名</label><input type="text" v-model="state.auth.username" class="text" placeholder="用于显示与管理员识别" /></div>
            <div class="row"><label>密码</label><input type="password" v-model="state.auth.password" class="text" /></div>
            <div class="row"><button class="submit" @click="onRegister">注册</button><button class="submit" style="background:#3b82f6" @click="onLogin">登录</button></div>
          </div>
          <div v-else>
            <div class="hint">当前用户</div>
            <div class="row"><input type="text" :value="state.auth.user.email || ''" readonly class="text" /></div>
            <div class="row"><input type="text" :value="state.auth.user.user_metadata?.username || ''" readonly class="text" /></div>
            <div class="row" v-if="state.auth.isAdmin"><span>管理员</span></div>
            <div class="row"><button class="submit" style="background:#ef4444" @click="onLogout">退出登录</button></div>
          </div>
        </div>
        <div class="divider"></div>
        <div class="hint">每隔多久开始这项计划</div>
        <div class="row">
          <label>选择日期</label>
          <input type="date" :min="minDate" :max="maxDate" v-model="state.startDateStr" />
        </div>
        <div class="row">
          <label>结束日期</label>
          <input type="date" :min="state.startDateStr" :max="maxDate" v-model="state.endDateStr" />
        </div>
        <div class="row inline">
          <label>每隔</label>
          <input type="number" min="1" v-model.number="state.intervalCount" class="num" />
          <select v-model="state.intervalUnit" class="unit">
            <option value="day">天</option>
            <option value="week">周</option>
            <option value="month">月</option>
          </select>
          <input type="text" v-model="state.eventLabel" placeholder="请输入计划文本" class="text" />
        </div>
        <div class="row">
          <button class="submit" @click="onSubmit">提交</button>
        </div>
        <div class="divider"></div>
        <div class="row">
          <button class="toggle" @click="state.showTasks = !state.showTasks">{{ state.showTasks ? '收起任务列表' : '展开任务列表' }}</button>
        </div>
        <div v-if="state.showTasks">
          <div class="task" v-for="t in state.tasks" :key="t.id">
            <div class="task-line">
              <span class="task-label">{{ t.label }}</span>
              <span class="task-range">{{ t.startDate }} → {{ t.endDate }}</span>
              <span class="task-interval">每隔 {{ t.intervalCount }} {{ unitLabel(t.intervalUnit) }}</span>
              <button class="del" @click="deleteTask(t.id)">删除</button>
            </div>
          </div>
          <div v-if="!state.tasks.length" class="task-empty">暂无任务</div>
        </div>
        <div class="divider"></div>
        <div class="hint">编辑选中日期</div>
        <div class="row">
          <label>选中日期</label>
          <input type="text" :value="state.editDate" readonly />
        </div>
        <div class="row">
          <label>文本</label>
          <input type="text" v-model="state.editText" class="text" placeholder="输入文本后自动保存" />
        </div>
        <div class="row">
          <label>标红</label>
          <input type="checkbox" v-model="state.editRed" />
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { reactive, onMounted, watch, onUnmounted } from 'vue'
import { createClient } from '@supabase/supabase-js'

const now = new Date()
const year = now.getFullYear()
const years = [year, year + 1]
const monthNames = ['一月','二月','三月','四月','五月','六月','七月','八月','九月','十月','十一月','十二月']
const weekdays = ['一','二','三','四','五','六','日']

const supaUrl = import.meta.env.VITE_SUPABASE_URL || ''
const supaKey = import.meta.env.VITE_SUPABASE_ANON_KEY || ''
const supa = (supaUrl && supaKey) ? createClient(supaUrl, supaKey) : null

function daysInMonth(y, m) {
  return new Date(y, m, 0).getDate()
}
function firstWeekday(y, m) {
  const w = new Date(y, m - 1, 1).getDay()
  return w === 0 ? 6 : w - 1
}
function pad(n) {
  return n < 10 ? '0' + n : '' + n
}
function fmt(y, m, d) {
  return `${y}-${pad(m)}-${pad(d)}`
}
function isWeekend(y, m, d) {
  const w = new Date(y, m - 1, d).getDay()
  return w === 0 || w === 6
}
function addRange(map, name, startStr, endStr) {
  const start = new Date(startStr)
  const end = new Date(endStr)
  let cur = new Date(start)
  while (cur <= end) {
    const k = fmt(cur.getFullYear(), cur.getMonth() + 1, cur.getDate())
    map.set(k, name)
    cur.setDate(cur.getDate() + 1)
  }
}
function mapOfficialHolidays(y) {
  const map = new Map()
  if (y === 2025) {
    addRange(map, '元旦', '2025-01-01', '2025-01-01')
    addRange(map, '春节', '2025-01-28', '2025-02-04')
    addRange(map, '清明节', '2025-04-04', '2025-04-06')
    addRange(map, '劳动节', '2025-05-01', '2025-05-05')
    addRange(map, '端午节', '2025-05-31', '2025-06-02')
    addRange(map, '国庆节', '2025-10-01', '2025-10-08')
    addRange(map, '中秋节', '2025-10-06', '2025-10-06')
  } else if (y === 2026) {
    addRange(map, '元旦', '2026-01-01', '2026-01-03')
    addRange(map, '春节', '2026-02-15', '2026-02-23')
    addRange(map, '清明节', '2026-04-04', '2026-04-06')
    addRange(map, '劳动节', '2026-05-01', '2026-05-05')
    addRange(map, '端午节', '2026-06-19', '2026-06-21')
    addRange(map, '中秋节', '2026-09-25', '2026-09-27')
    addRange(map, '国庆节', '2026-10-01', '2026-10-07')
  }
  return map
}
function holidayLabel(y, m, d) {
  const map = mapOfficialHolidays(y)
  return map.get(fmt(y, m, d)) || ''
}
function officialAdjustWorkdays(y) {
  const set = new Set()
  if (y === 2025) {
    set.add('2025-01-26')
    set.add('2025-02-08')
    set.add('2025-04-27')
    set.add('2025-09-28')
    set.add('2025-10-11')
  } else if (y === 2026) {
    set.add('2026-01-04')
    set.add('2026-02-14')
    set.add('2026-02-28')
    set.add('2026-05-09')
    set.add('2026-09-20')
    set.add('2026-10-10')
  }
  return set
}
function isAdjustWorkday(y, m, d) {
  const set = officialAdjustWorkdays(y)
  return set.has(fmt(y, m, d))
}

const state = reactive({
  gen: {},
  manual: {},
  cellScale: 1.35,
  done: {},
  tasks: [],
  auth: { email: '', password: '', username: '', user: null, isAdmin: false },
  startDateStr: (() => {
    const mm = pad(now.getMonth() + 1)
    const dd = pad(now.getDate())
    return `${year}-${mm}-${dd}`
  })(),
  endDateStr: (() => {
    return `${year}-12-31`
  })(),
  intervalCount: 1,
  intervalUnit: 'day',
  eventLabel: '',
  editDate: (() => {
    const mm = pad(now.getMonth() + 1)
    const dd = pad(now.getDate())
    return `${year}-${mm}-${dd}`
  })(),
  editText: '',
  editRed: false,
  showTasks: false
})
state.panel = {
  x: window.innerWidth - 420,
  y: Math.max(24, Math.floor(window.innerHeight * 0.15)),
  scale: 1.25,
  minimized: false,
  dragging: false,
  dx: 0,
  dy: 0
}

const minDate = `${year}-01-01`
const maxDate = `${year + 1}-12-31`

function clampToRange(date) {
  const d = new Date(date)
  const start = new Date(year, 0, 1)
  const end = new Date(year + 1, 11, 31)
  if (d < start) return start
  if (d > end) return end
  return d
}
function addByUnit(date, count, unit) {
  const d = new Date(date)
  if (unit === 'day') d.setDate(d.getDate() + count)
  else if (unit === 'week') d.setDate(d.getDate() + count * 7)
  else if (unit === 'month') {
    const day = d.getDate()
    const targetMonth = d.getMonth() + count
    const tmp = new Date(d.getFullYear(), targetMonth + 1, 0)
    const maxDay = tmp.getDate()
    const clampedDay = Math.min(day, maxDay)
    d.setMonth(targetMonth)
    d.setDate(clampedDay)
  }
  return d
}

function onSubmit() {
  const label = state.eventLabel.trim()
  if (!label) return
  const start = clampToRange(state.startDateStr)
  const end = clampToRange(state.endDateStr)
  let cur = start
  const count = Math.max(1, Number(state.intervalCount) || 1)
  const unit = state.intervalUnit
  const task = {
    id: Date.now() + Math.random().toString(16).slice(2),
    startDate: fmt(start.getFullYear(), start.getMonth() + 1, start.getDate()),
    endDate: fmt(end.getFullYear(), end.getMonth() + 1, end.getDate()),
    intervalCount: count,
    intervalUnit: unit,
    label
  }
  state.tasks.push(task)
  persistTasks()
  recomputeGenerated()
}
function eventsList(y, m, d) {
  const k = fmt(y, m, d)
  const list = []
  const em = state.manual[k]
  if (em) list.push({ key: `m-${k}`, text: typeof em === 'string' ? em : (em.text || ''), red: typeof em === 'object' ? !!em.red : false })
  const eg = state.gen[k] || []
  for (const item of eg) list.push({ key: `g-${item.id}-${k}`, text: item.text, red: false })
  return list.sort((a, b) => a.red === b.red ? a.text.localeCompare(b.text, 'zh') : (a.red ? -1 : 1))
}
function eventsCount(y, m, d) {
  return eventsList(y, m, d).length
}
function cellLines(y, m, d) {
  let lines = 0
  if (holidayLabel(y, m, d)) lines++
  if (isAdjustWorkday(y, m, d)) lines++
  lines += eventsCount(y, m, d)
  return lines
}
function monthCellSize(y, m) {
  let maxLines = 0
  const dimBase = 64
  const perLine = 16
  for (let d = 1; d <= daysInMonth(y, m); d++) {
    maxLines = Math.max(maxLines, cellLines(y, m, d))
  }
  return dimBase + maxLines * perLine
}
function yearCellSize(y) {
  let maxSize = 64
  for (let m = 1; m <= 12; m++) {
    maxSize = Math.max(maxSize, monthCellSize(y, m))
  }
  return Math.ceil(maxSize * state.cellScale)
}
function dayClass(y, m, d) {
  const cls = []
  const wk = isWeekend(y, m, d)
  if (wk && !isAdjustWorkday(y, m, d)) cls.push('weekend')
  if (isAdjustWorkday(y, m, d)) cls.push('workday-adjust')
  if (holidayLabel(y, m, d)) cls.push('holiday')
  if (eventsCount(y, m, d) > 0) cls.push('has-event')
  return cls.join(' ')
}
function isSelected(y, m, d) {
  return state.editDate === fmt(y, m, d)
}
function onSelectDay(y, m, d) {
  const k = fmt(y, m, d)
  state.editDate = k
  const e = state.manual[k]
  state.editText = typeof e === 'string' ? e : (e && e.text) || ''
  state.editRed = typeof e === 'object' ? !!e.red : false
}
function persistManual() {
  if (state.auth.user && supa) saveRemote()
  else try { localStorage.setItem('rili-manual', JSON.stringify(state.manual)) } catch {}
}
function persistTasks() {
  if (state.auth.user && supa) saveRemote()
  else try { localStorage.setItem('rili-tasks', JSON.stringify(state.tasks)) } catch {}
}
function recomputeGenerated() {
  const gen = {}
  const toKey = (d) => fmt(d.getFullYear(), d.getMonth() + 1, d.getDate())
  for (const t of state.tasks) {
    let cur = new Date(t.startDate)
    const end = new Date(t.endDate)
    while (cur <= end) {
      const k = toKey(cur)
      if (!gen[k]) gen[k] = []
      gen[k].push({ id: t.id, text: t.label })
      cur = addByUnit(cur, t.intervalCount, t.intervalUnit)
      if (cur <= new Date(t.startDate)) break
    }
  }
  state.gen = gen
}
function deleteTask(id) {
  state.tasks = state.tasks.filter(t => t.id !== id)
  persistTasks()
  recomputeGenerated()
}
function isDone(key) { return !!state.done[key] }
function toggleDone(key, checked) { if (checked) state.done[key] = true; else delete state.done[key]; persistDone() }
function persistDone() { if (state.auth.user && supa) saveRemote(); else try { localStorage.setItem('rili-done', JSON.stringify(state.done)) } catch {} }
async function saveRemote() {
  try {
    const user = state.auth.user
    if (!user || !supa) return
    const payload = { user_id: user.id, manual: state.manual, tasks: state.tasks, done: state.done }
    await supa.from('rili_data').upsert(payload, { onConflict: 'user_id' })
  } catch {}
}
async function loadRemote() {
  try {
    const user = state.auth.user
    if (!user || !supa) return
    const { data } = await supa.from('rili_data').select('manual,tasks,done').eq('user_id', user.id).single()
    if (data) {
      state.manual = data.manual || {}
      state.tasks = Array.isArray(data.tasks) ? data.tasks : []
      state.done = data.done || {}
      recomputeGenerated()
    }
  } catch {}
}
async function onRegister() {
  if (!supa) return
  const email = (state.auth.email || '').trim()
  const password = (state.auth.password || '').trim()
  const username = (state.auth.username || '').trim()
  if (!email || !password || !username) return
  const { data } = await supa.auth.signUp({ email, password, options: { data: { username } } })
  if (data && data.user) {
    state.auth.user = data.user
    state.auth.isAdmin = (data.user.user_metadata && data.user.user_metadata.username === 'pyq')
    await loadRemote()
  }
}
async function onLogin() {
  if (!supa) return
  const email = (state.auth.email || '').trim()
  const password = (state.auth.password || '').trim()
  if (!email || !password) return
  const { data } = await supa.auth.signInWithPassword({ email, password })
  if (data && data.user) {
    state.auth.user = data.user
    state.auth.isAdmin = (data.user.user_metadata && data.user.user_metadata.username === 'pyq')
    await loadRemote()
  }
}
async function onLogout() {
  if (!supa) return
  await supa.auth.signOut()
  state.auth.user = null
  state.auth.isAdmin = false
}
watch(() => [state.editDate, state.editText, state.editRed], () => {
  const k = state.editDate
  const text = (state.editText || '').trim()
  const red = !!state.editRed
  if (!k) return
  if (!text && !red) {
    delete state.manual[k]
  } else {
    state.manual[k] = { text, red }
  }
  persistManual()
})
onMounted(() => {
  if (supa) {
    supa.auth.onAuthStateChange(async (_, session) => {
      const u = session && session.user
      state.auth.user = u || null
      state.auth.isAdmin = !!(u && u.user_metadata && u.user_metadata.username === 'pyq')
      if (u) await loadRemote()
    })
  }
  try {
    const savedManual = localStorage.getItem('rili-manual')
    if (savedManual) {
      const obj = JSON.parse(savedManual)
      if (obj && typeof obj === 'object') state.manual = obj
    }
    const savedTasks = localStorage.getItem('rili-tasks')
    if (savedTasks) {
      const arr = JSON.parse(savedTasks)
      if (Array.isArray(arr)) state.tasks = arr
    }
    const savedDone = localStorage.getItem('rili-done')
    if (savedDone) {
      const obj = JSON.parse(savedDone)
      if (obj && typeof obj === 'object') state.done = obj
    }
  } catch {}
  recomputeGenerated()
  const todayKey = fmt(year, now.getMonth() + 1, now.getDate())
  state.editDate = todayKey
  const e = state.manual[todayKey]
  state.editText = typeof e === 'string' ? e : (e && e.text) || ''
  state.editRed = typeof e === 'object' ? !!e.red : false
  onSelectDay(year, now.getMonth() + 1, now.getDate())
  setTimeout(() => {
    const el = document.querySelector(`[data-date="${todayKey}"]`)
    if (el) el.scrollIntoView({ behavior: 'smooth', block: 'center' })
  }, 0)
  const onMove = (e) => {
    if (!state.panel.dragging) return
    state.panel.x = e.clientX - state.panel.dx
    state.panel.y = e.clientY - state.panel.dy
  }
  const onUp = () => { state.panel.dragging = false }
  window.addEventListener('mousemove', onMove)
  window.addEventListener('mouseup', onUp)
  state.panel._onMove = onMove
  state.panel._onUp = onUp
})
onUnmounted(() => {
  if (state.panel._onMove) window.removeEventListener('mousemove', state.panel._onMove)
  if (state.panel._onUp) window.removeEventListener('mouseup', state.panel._onUp)
})
watch(() => state.startDateStr, (val) => {
  try {
    const d = new Date(val)
    const y = d.getFullYear()
    state.endDateStr = `${y}-12-31`
  } catch {}
})
function unitLabel(u) {
  return u === 'day' ? '天' : (u === 'week' ? '周' : '月')
}
function startDragPanel(e) {
  state.panel.dragging = true
  const rect = e.currentTarget.parentElement.getBoundingClientRect()
  state.panel.dx = e.clientX - rect.left
  state.panel.dy = e.clientY - rect.top
}
function zoomIn() { state.panel.scale = Math.min(1.8, +(state.panel.scale + 0.1).toFixed(2)) }
function zoomOut() { state.panel.scale = Math.max(0.7, +(state.panel.scale - 0.1).toFixed(2)) }
function resetPanel() {
  state.panel.scale = 1.25
  state.panel.x = window.innerWidth - 420
  state.panel.y = Math.max(24, Math.floor(window.innerHeight * 0.15))
}
function scrollToToday() {
  const t = new Date()
  const y = t.getFullYear()
  const m = t.getMonth() + 1
  const d = t.getDate()
  onSelectDay(y, m, d)
  const key = fmt(y, m, d)
  const el = document.querySelector(`[data-date="${key}"]`)
  if (el) el.scrollIntoView({ behavior: 'smooth', block: 'center' })
}
</script>

<style scoped>
.layout { --panel-width: 520px; --panel-gap: 24px; }
.h1 { margin: 0 0 12px; }
.calendar-year { display: grid; grid-template-columns: repeat(2, auto); gap: 16px; justify-content: center; justify-items: center; }
.month { border: 1px solid #e5e7eb; border-radius: 8px; padding: 8px; background: #fff; }
.month-header { font-weight: 600; padding: 8px; text-align: center; }
.weekdays { display: grid; grid-template-columns: repeat(7, 1fr); gap: 4px; padding: 0 8px; }
.weekday { font-size: 13px; color: #374151; text-align: center; }
.days { display: grid; grid-template-columns: repeat(7, var(--cell-size)); gap: 4px; padding: 8px; }
.day { width: var(--cell-size); height: var(--cell-size); border: 2px solid #d1d5db; border-radius: 8px; padding: 8px; position: relative; box-sizing: border-box; overflow: hidden; }
.day.placeholder { visibility: hidden; }
.date { font-size: 14px; color: #111827; }
.marks { margin-top: 4px; display: flex; flex-direction: column; gap: 2px; overflow-wrap: anywhere; }
.mark { font-size: 12px; color: #ef4444; }
.workday { font-size: 12px; color: #2563eb; }
.events { display: flex; flex-direction: column; gap: 2px; }
.event { font-size: 13px; color: #1f2937; display: flex; align-items: center; gap: 6px; }
.event.done .event-text { text-decoration: line-through; color: #9ca3af; }
.event-check { width: 14px; height: 14px; }
.event-number { font-size: 12px; color: #6b7280; }
.event-text { word-break: break-word; overflow-wrap: anywhere; }
.event.red { color: #ef4444; }
.day.weekend { background: #fafafa; }
.day.holiday { box-shadow: inset 0 0 0 2px #fecaca; }
.day.workday-adjust { background: #fffbe6; border-color: #fde68a; }
.day.has-event { background: #f0f9ff; border-color: #bae6fd; }
.day.selected { box-shadow: inset 0 0 0 2px #60a5fa; background: #eef6ff; border-color: #93c5fd; }
.side-panel { position: fixed; left: 0; top: 0; transform: translate(var(--panel-x), var(--panel-y)) scale(var(--panel-scale)); transform-origin: top left; width: var(--panel-width); background: #fff; border: 1px solid #e5e7eb; border-radius: 8px; padding: 16px; box-shadow: 0 10px 30px rgba(0,0,0,0.12); z-index: 1000; }
.panel-header { display: flex; align-items: center; justify-content: space-between; margin: -8px -8px 12px; padding: 8px; border-bottom: 1px solid #f3f4f6; cursor: move; }
.panel-header .title { font-weight: 600; }
.panel-header .actions { display: flex; gap: 8px; }
.panel-header .btn { padding: 4px 8px; border: 1px solid #e5e7eb; border-radius: 6px; background: #fff; cursor: pointer; }
.hint { font-weight: 600; margin-bottom: 12px; }
.row { display: flex; align-items: center; gap: 8px; margin-bottom: 12px; flex-wrap: wrap; }
.row.inline label { margin-right: 4px; }
.num { width: 80px; }
.unit { width: 100px; }
.text { flex: 1; }
.submit { background: #10b981; color: #fff; border: none; padding: 8px 16px; border-radius: 6px; cursor: pointer; }
.submit:hover { opacity: .9 }
.divider { height: 1px; background: #e5e7eb; margin: 12px 0; }

@media (max-width: 1280px) {
  .side-panel { width: min(92vw, 560px); }
}
@media (max-width: 1400px) {
  .calendar-year { grid-template-columns: repeat(2, auto); }
}
@media (max-width: 900px) {
  .calendar-year { grid-template-columns: auto; }
}
</style>
