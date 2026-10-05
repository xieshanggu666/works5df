<script setup>
import { ref, computed, reactive } from 'vue'
import { useEventStore } from '@/store/event'
const store = useEventStore()

const toast = ref('')
function flash(msg, ok = true) { toast.value = (ok ? '' : '⚠️ ') + msg; setTimeout(() => toast.value = '', 4000) }

// 当前操作角色（演示协同：医疗/安保/裁判/组委会，权限前后端一致）
const ROLES = {
  medical: { icon: '🩺', text: '医疗', color: '#22c15e' },
  security: { icon: '🛡️', text: '安保', color: '#2f9bff' },
  referee: { icon: '🧑‍⚖️', text: '裁判', color: '#ff7a2f' },
  committee: { icon: '🏛️', text: '组委会', color: '#9b59b6' }
}
const actorRole = ref('committee')
const actorName = reactive({ medical: '赵医生', security: '钱安保', referee: '张裁判', committee: '王总指挥' })
const actor = () => ({ actor_role: actorRole.value, actor_name: actorName[actorRole.value] })
const isCommittee = computed(() => actorRole.value === 'committee')

const CATEGORIES = {
  injury: { icon: '🩹', text: '运动伤病' }, medical: { icon: '🚑', text: '医疗急救' },
  security: { icon: '🚨', text: '治安冲突' }, crowd: { icon: '👥', text: '观众秩序' },
  facility: { icon: '🛠️', text: '场地设施' }, weather: { icon: '⛈️', text: '极端天气' },
  doping: { icon: '💊', text: '违规/兴奋剂' }, other: { icon: '📌', text: '其他' }
}
const SEVERITY = {
  high: { icon: '🔴', text: '重大', tag: 'r' },
  medium: { icon: '🟠', text: '较大', tag: 'o' },
  low: { icon: '🟢', text: '一般', tag: 'g' }
}
const STATUS = {
  reported: { text: '待分级', tag: 'o' }, triaged: { text: '处置中', tag: 'b' },
  handling: { text: '处置中', tag: 'b' }, resolved: { text: '已解决待结案', tag: 'y' },
  closed: { text: '已结案', tag: 'gray' }
}
const LOG_META = {
  report: { text: '上报', tag: 'o' }, triage: { text: '受理分级', tag: 'b' },
  handle: { text: '处置进展', tag: 'b' }, resolve: { text: '标记解决', tag: 'y' },
  close: { text: '结案', tag: 'g' }, reopen: { text: '重新开启', tag: 'r' },
  badge_hold: { text: '证件暂扣', tag: 'r' }, badge_release: { text: '解除暂扣', tag: 'g' },
  match_pause: { text: '场次暂停', tag: 'r' }, match_resume: { text: '恢复场次', tag: 'g' },
  match_reschedule: { text: '暂停改期', tag: 'y' }
}

const counts = computed(() => {
  const c = { reported: 0, active: 0, resolved: 0, closed: 0, high: 0, paused: new Set(), held: 0 }
  store.incidents.forEach(i => {
    if (i.status === 'reported') c.reported++
    if (['triaged', 'handling'].includes(i.status)) c.active++
    if (i.status === 'resolved') c.resolved++
    if (i.status === 'closed') c.closed++
    if (i.status !== 'closed' && i.severity === 'high') c.high++
    i.paused_matches?.forEach(m => c.paused.add(m.id))
    c.held += i.held_badges?.length || 0
  })
  c.paused = c.paused.size
  return c
})

/* ---------- 上报 ---------- */
const showReport = ref(false)
const form = reactive({
  title: '', category: 'injury', detail: '', sport_id: '', venue_id: '', match_id: '',
  badge_id: '', badge_reason: '', location: '', contact: ''
})
function resetForm() {
  Object.assign(form, { title: '', category: 'injury', detail: '', sport_id: '', venue_id: '', match_id: '', badge_id: '', badge_reason: '', location: '', contact: '' })
}
// 选择项目后带出场地；场次限定为该项目的待赛场
function pickSport() {
  form.match_id = ''
  const s = store.sports.find(x => x.id === Number(form.sport_id))
  if (s) { const v = store.venues.find(v => v.name === s.venue); if (v) form.venue_id = String(v.id) }
}
const formMatches = computed(() => {
  const list = store.matches.filter(m => m.status === 'scheduled' && !m.is_paused)
  return form.sport_id ? list.filter(m => m.sport_id === Number(form.sport_id)) : list
})
function matchText(m) {
  const sp = store.sports.find(s => s.id === m.sport_id)?.name || ''
  return `${sp} · ${m.stage}${m.group_name || ''} ${m.teamA?.name || '待定'}VS${m.teamB?.name || '待定'}（${m.time_label}）`
}
async function submit() {
  if (!form.title.trim()) return flash('请填写事件标题')
  if (!form.detail.trim()) return flash('请描述事件经过')
  try {
    const r = await store.submitIncident({
      title: form.title.trim(), category: form.category, detail: form.detail.trim(),
      sport_id: form.sport_id ? Number(form.sport_id) : null,
      venue_id: form.venue_id ? Number(form.venue_id) : null,
      match_id: form.match_id ? Number(form.match_id) : null,
      badge_id: form.badge_id ? Number(form.badge_id) : null,
      badge_reason: form.badge_reason.trim() || null,
      location: form.location.trim(), contact: form.contact.trim(), ...actor()
    })
    flash(`✅ 事件已上报（编号 ${r.code}），等待组委会分级`)
    resetForm(); showReport.value = false
  } catch (e) { flash(e.message, false) }
}

/* ---------- 列表筛选 ---------- */
const filter = ref('open')
const list = computed(() => store.incidents.filter(i => {
  if (filter.value === 'all') return true
  if (filter.value === 'open') return i.status !== 'closed'
  return i.status === filter.value
}))

/* ---------- 联动操作面板 ---------- */
const panel = reactive({ id: null, mode: '', mid: null, text: '', time_label: '', venue_id: '', error: '', conflicts: null })
function openPanel(i, mode, mid = null) {
  panel.id = i.id; panel.mode = mode; panel.mid = mid; panel.text = ''; panel.error = ''; panel.conflicts = null
  if (mode === 'reschedule' && mid) {
    const m = store.matches.find(x => x.id === mid)
    panel.time_label = m?.time_label || ''; panel.venue_id = m?.venue_id ? String(m.venue_id) : ''
  }
}
function closePanel() { panel.id = null; panel.mode = ''; panel.mid = null }

async function triage(i, severity) {
  const note = prompt(`${SEVERITY[severity].text}事件分级研判意见（必填）：`, severity === 'high' ? '启动重大事件应急响应' : severity === 'medium' ? '多部门协同处置' : '常规处置，密切跟踪')
  if (note == null) return
  if (!note.trim()) return flash('分级研判意见不能为空')
  try { await store.triageIncident(i.id, { severity, triage_note: note.trim(), ...actor() }); flash(`✅ 已分级为${SEVERITY[severity].text}事件，进入处置`) }
  catch (e) { flash(e.message, false) }
}
async function addProgress(i, resolved = false) {
  const content = prompt(resolved ? '处置结果说明（标记已解决，等待组委会结案）：' : '处置进展（必填，如伤情、处置措施、现场情况）：')
  if (content == null) return
  if (!content.trim()) return flash('处置内容不能为空')
  try {
    await store.handleIncident(i.id, { content: content.trim(), status: resolved ? 'resolved' : 'handling', ...actor() })
    flash(resolved ? '✅ 事件已标记解决，等待组委会结案' : '✅ 处置进展已记录并同步协同各方')
  } catch (e) { flash(e.message, false) }
}
async function closeCase(i) {
  const resolution = prompt('结案总结（处置结果与后续防范，必填）：')
  if (resolution == null) return
  if (!resolution.trim()) return flash('结案总结不能为空')
  try {
    const r = await store.closeIncident(i.id, { resolution: resolution.trim(), release_badges: true, ...actor() })
    flash(`✅ 事件 ${i.code} 已结案${r.released_badges?.length ? `，随案解除暂扣证件 ${r.released_badges.length} 张` : ''}`)
  } catch (e) { flash(e.message, false) }
}
async function reopen(i) {
  const note = prompt('重新开启原因（如遗留问题/复发）：', '现场反馈问题仍需跟进')
  if (note == null) return
  try { await store.reopenIncident(i.id, { note, ...actor() }); flash('事件已重新开启，回到处置中') }
  catch (e) { flash(e.message, false) }
}
async function confirmHold(i) {
  if (!form2.badge_id) return flash('请选择要暂扣的证件')
  try {
    await store.holdBadgeForIncident(i.id, Number(form2.badge_id), panel.text.trim() || `${i.code} 事件暂扣`, actor())
    flash('✅ 证件已暂扣并关联本事件，入场将被拦截')
    closePanel()
  } catch (e) { flash(e.message, false) }
}
async function releaseBadge(i, b) {
  try { await store.releaseBadgeForIncident(i.id, b.badge_id, '核查后提前解除暂扣', actor()); flash(`✅ 证件 ${b.code} 已解除暂扣`) }
  catch (e) { flash(e.message, false) }
}
async function confirmPause(i) {
  if (!form2.match_id) return flash('请选择要暂停的场次')
  try {
    await store.pauseMatchForIncident(i.id, Number(form2.match_id), panel.text.trim() || `${i.code} 安全事件暂停`, actor())
    flash('✅ 场次已暂停：入场核验将拦截，成绩不可录入')
    closePanel()
  } catch (e) { flash(e.message, false) }
}
async function resumeMatch(i, m) {
  const note = prompt('恢复说明（选填）：', '现场恢复，比赛继续')
  if (note == null) return
  try { await store.resumeMatchForIncident(i.id, m.id, note.trim(), actor()); flash('✅ 场次已恢复正常待赛，入场恢复放行') }
  catch (e) { flash(e.message, false) }
}
async function confirmReschedule(i) {
  if (!panel.time_label) return flash('请填写改期时间')
  try {
    const r = await store.rescheduleMatchForIncident(i.id, panel.mid, {
      time_label: panel.time_label, venue_id: panel.venue_id ? Number(panel.venue_id) : null,
      reason: panel.text.trim() || '安全事件暂停改期', ...actor()
    })
    flash(`✅ 已暂停改期至 ${panel.time_label}：自动重排 ${r.rearranged?.length || 0} 席、补齐 ${r.auto_filled?.length || 0} 席`)
    closePanel()
  } catch (e) {
    if (e.status === 409) { panel.conflicts = e.conflicts; panel.error = e.message }
    else flash(e.message, false)
  }
}
async function forceReschedule(i) {
  try {
    const r = await store.rescheduleMatchForIncident(i.id, panel.mid, {
      time_label: panel.time_label, venue_id: panel.venue_id ? Number(panel.venue_id) : null,
      reason: panel.text.trim() || '安全事件暂停改期（强制）', force: true, ...actor()
    })
    flash(`✅ 已强制改期至 ${panel.time_label}`)
    closePanel()
  } catch (e) { flash(e.message, false) }
}

// 面板内选择的证件/场次（hold/pause 共用）
const form2 = reactive({ badge_id: '', match_id: '' })
function openPanel2(i, mode) { form2.badge_id = ''; form2.match_id = ''; openPanel(i, mode) }
const panelMatches = computed(() => store.matches.filter(m => m.status === 'scheduled' && !m.is_paused))
const SUBJECT_TEXT = { team: '队伍', athlete: '田径运动员', referee: '裁判', staff: '场地工作人员' }

const unitColor = uid => store.unitOfUid(uid)?.color || '#ccc'
const catIcon = c => CATEGORIES[c]?.icon || '📌'
function pausedOf(i) { return store.matches.filter(m => m.is_paused && m.paused_incident_id === i.id) }
</script>

<template>
  <div v-if="store.loaded">
    <div class="page-h">
      <div><h2>🚨 赛事安全事件处置</h2><div class="sub">医疗 · 安保 · 裁判 · 组委会协同上报、分级、处置、结案 · 联动证件暂扣、场次暂停改期与异常审计</div></div>
      <div class="row">
        <select v-model="actorRole" class="role-switch">
          <option v-for="(r, k) in ROLES" :key="k" :value="k">{{ r.icon }} 我是{{ r.text }}</option>
        </select>
        <input v-model="actorName[actorRole]" style="width:110px" />
        <button class="btn primary" @click="showReport = !showReport">📣 上报事件</button>
      </div>
    </div>

    <div v-if="toast" class="toast">{{ toast }}</div>

    <!-- 统计卡 -->
    <div class="grid" style="grid-template-columns:repeat(6,1fr)">
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#ff7a2f,#ffb27e)"></span><span class="ic">📥</span><b>{{ counts.reported }}</b><em>待分级</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#2f9bff,#79c4ff)"></span><span class="ic">🔧</span><b>{{ counts.active }}</b><em>处置中</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#ffb92b,#ffd98a)"></span><span class="ic">✔️</span><b>{{ counts.resolved }}</b><em>已解决待结案</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#90a4ae,#c3ccd6)"></span><span class="ic">📁</span><b>{{ counts.closed }}</b><em>已结案</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#e5484d,#f0a1a1)"></span><span class="ic">🔴</span><b>{{ counts.high }}</b><em>未结重大事件</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#8e44ad,#c39bd3)"></span><span class="ic">⏸️</span><b>{{ counts.paused }}<span class="ph" style="font-size:15px"> / 🪪{{ counts.held }}</span></b><em>暂停场次 / 暂扣证件</em></div>
    </div>

    <!-- 上报表单 -->
    <transition name="slide">
      <div v-if="showReport" class="card mt">
        <div class="caption"><span>📣 协同上报安全事件 <span class="hint">当前身份：{{ ROLES[actorRole].icon }} {{ actorName[actorRole] }}（{{ ROLES[actorRole].text }}）</span></span></div>
        <div class="pad">
          <div class="grid g2" style="gap:12px 18px">
            <div class="form-row"><label>事件标题</label><input v-model="form.title" placeholder="一句话概括，如：球员拼抢受伤 / 看台冲突" /></div>
            <div class="form-row"><label>事件类型</label>
              <div class="filters">
                <button v-for="(c, k) in CATEGORIES" :key="k" type="button" class="chip" :class="{ on: form.category === k }" @click="form.category = k">{{ c.icon }} {{ c.text }}</button>
              </div>
            </div>
            <div class="form-row"><label>关联项目</label>
              <select v-model="form.sport_id" @change="pickSport"><option value="">不关联项目</option><option v-for="s in store.sports" :key="s.id" :value="s.id">{{ s.name }}</option></select>
            </div>
            <div class="form-row"><label>关联场地</label>
              <select v-model="form.venue_id"><option value="">不指定</option><option v-for="v in store.venues" :key="v.id" :value="v.id">{{ v.name }}</option></select>
            </div>
            <div class="form-row"><label>关联场次</label>
              <select v-model="form.match_id"><option value="">不关联场次</option><option v-for="m in formMatches" :key="m.id" :value="m.id">{{ matchText(m) }}</option></select>
            </div>
            <div class="form-row"><label>具体位置</label><input v-model="form.location" placeholder="如：五人足球场东侧边线 / 二层3号看台" /></div>
            <div class="form-row"><label>当事人/联系</label><input v-model="form.contact" placeholder="当事人或联系人（选填）" /></div>
            <div class="form-row"><label>上报即暂扣</label>
              <div style="flex:1;display:flex;gap:8px">
                <select v-model="form.badge_id" style="flex:1"><option value="">不暂扣证件</option><option v-for="b in store.badges" :key="b.id" :value="b.id">{{ b.code }} · {{ b.name }}（{{ SUBJECT_TEXT[b.subject_type] }}）</option></select>
              </div>
            </div>
          </div>
          <div class="form-row" style="align-items:flex-start;margin-top:4px" v-if="form.badge_id">
            <label>暂扣原因</label><input v-model="form.badge_reason" placeholder="如：涉事人员证件暂扣待核（事件处置期间不同步解停）" />
          </div>
          <div class="form-row" style="align-items:flex-start;margin-top:4px">
            <label style="margin-top:7px">事件经过</label>
            <textarea v-model="form.detail" rows="3" placeholder="详细描述：时间、人员、经过、现场已采取措施" style="flex:1;resize:vertical"></textarea>
          </div>
          <div class="row mt8" style="justify-content:flex-end">
            <button class="btn ghost sm" @click="showReport = false">取消</button>
            <button class="btn primary" @click="submit">📣 提交上报</button>
          </div>
        </div>
      </div>
    </transition>

    <!-- 流程说明 -->
    <div class="card mt">
      <div class="pad rules">
        <b style="font-size:12.5px">🔄 协同处置流程：</b>
        <span class="tag o">四方上报 → 待分级</span>
        <span class="ph">→</span>
        <span class="tag b">🏛️ 组委会分级（一般/较大/重大）</span>
        <span class="ph">→</span>
        <span class="tag b">🩺🛡️🧑‍⚖️ 多部门处置进展</span>
        <span class="ph">→</span>
        <span class="tag y">标记已解决</span>
        <span class="ph">→</span>
        <span class="tag g">组委会结案</span>
        <span class="ph" style="margin-left:14px">联动：</span>
        <span class="tag r">🪪 证件暂扣</span><span class="ph">事件处置期间不自动解停</span>
        <span class="tag r">⏸️ 场次暂停</span><span class="ph">核验拒绝、比分拦截</span>
        <span class="tag y">🕐 暂停改期</span><span class="ph">冲突自动重排，全程入异常审计</span>
      </div>
    </div>

    <!-- 事件案卷 -->
    <div class="card mt">
      <div class="caption">
        <span>🗂️ 事件案卷 <span class="hint">共 {{ list.length }} 件</span></span>
        <div class="filters">
          <button class="chip" :class="{ on: filter === 'open' }" @click="filter = 'open'">处置中</button>
          <button class="chip" :class="{ on: filter === 'reported' }" @click="filter = 'reported'">待分级</button>
          <button class="chip" :class="{ on: filter === 'resolved' }" @click="filter = 'resolved'">待结案</button>
          <button class="chip" :class="{ on: filter === 'closed' }" @click="filter = 'closed'">已结案</button>
          <button class="chip" :class="{ on: filter === 'all' }" @click="filter = 'all'">全部</button>
        </div>
      </div>
      <div class="pad" style="display:flex;flex-direction:column;gap:14px">
        <div v-for="i in list" :key="i.id" class="icard" :class="i.status">
          <!-- 头部 -->
          <div class="iheader">
            <span>
              <b class="mono" style="color:var(--accent3)">{{ i.code }}</b>
              <span class="tag" style="margin-left:8px" :class="SEVERITY[i.severity]?.tag || 'gray'">
                {{ i.severity ? SEVERITY[i.severity].icon + SEVERITY[i.severity].text : '未分级' }}
              </span>
              <b style="margin-left:8px">{{ catIcon(i.category) }} {{ i.title }}</b>
            </span>
            <span class="tag" :class="STATUS[i.status].tag">{{ STATUS[i.status].text }}</span>
          </div>

          <!-- 元信息 -->
          <div class="meta-line">
            <span class="tag" :style="{ background: ROLES[i.reporter_role].color + '1a', color: ROLES[i.reporter_role].color }">{{ ROLES[i.reporter_role].icon }} {{ i.reporter_name }}（{{ ROLES[i.reporter_role].text }}上报）</span>
            <span v-if="i.sport">🏅 {{ i.sport.name }}</span>
            <span v-if="i.venue">📍 {{ i.venue.name }}</span>
            <span v-if="i.location">📌 {{ i.location }}</span>
            <span v-if="i.contact">👤 {{ i.contact }}</span>
            <span>{{ i.created_at }}</span>
          </div>
          <div class="detail-line">💬 {{ i.detail }}</div>
          <div v-if="i.triage_note" class="triage-box">
            <b>🏛️ 分级研判（{{ i.triage_name }} · {{ i.triaged_at }}）：</b>{{ i.triage_note }}
          </div>
          <div v-if="i.resolution" class="resolution-box"><b>📁 结案总结（{{ i.close_name }} · {{ i.closed_at }}）：</b>{{ i.resolution }}</div>

          <!-- 联动状态：暂停场次 / 暂扣证件 -->
          <div v-if="pausedOf(i).length || i.held_badges?.length" class="link-box">
            <div v-if="pausedOf(i).length" v-for="m in pausedOf(i)" :key="'pm'+m.id" class="link-row">
              <span class="tag r">⏸️ 已暂停</span>
              <b>{{ matchText(m) }}</b>
              <span class="ph">{{ m.pause_reason }}</span>
              <span v-if="i.status !== 'closed'" class="row" style="margin-left:auto">
                <button class="mini" @click="openPanel(i, 'reschedule', m.id)">🕐 暂停改期</button>
                <button class="mini ok" @click="resumeMatch(i, m)">▶️ 恢复</button>
              </span>
            </div>
            <div v-for="b in i.held_badges" :key="'hb'+b.id" class="link-row">
              <span class="tag r">🪪 暂扣中</span>
              <b class="mono">{{ b.code }}</b><span>{{ b.name }}（{{ SUBJECT_TEXT[b.subject_type] }}）</span>
              <span class="ph">{{ b.block_reason }}</span>
              <span v-if="i.status !== 'closed'" style="margin-left:auto">
                <button class="mini ok" @click="releaseBadge(i, { badge_id: b.id, code: b.code })">解除暂扣</button>
              </span>
            </div>
          </div>

          <!-- 操作区 -->
          <div v-if="i.status !== 'closed'" class="row mt8" style="gap:6px;justify-content:flex-end;flex-wrap:wrap">
            <template v-if="i.status === 'reported'">
              <button v-if="isCommittee" class="btn sm" style="background:#e6f7ec;color:var(--accent2)" @click="triage(i, 'low')">🟢 一般</button>
              <button v-if="isCommittee" class="btn sm" style="background:#fff1e6;color:var(--accent)" @click="triage(i, 'medium')">🟠 较大</button>
              <button v-if="isCommittee" class="btn sm" style="background:#ffecec;color:#e5484d" @click="triage(i, 'high')">🔴 重大</button>
              <span v-else class="tag gray" style="margin-right:auto">🏛️ 等待组委会分级</span>
            </template>
            <template v-else>
              <button class="btn ghost sm" @click="addProgress(i, false)">📝 处置进展</button>
              <button v-if="i.status !== 'resolved'" class="btn sm" style="background:#fff6dd;color:#c98a00" @click="addProgress(i, true)">✔️ 标记解决</button>
            </template>
            <button class="btn sm" style="background:#ffecec;color:#e5484d" @click="openPanel2(i, 'hold')">🪪 暂扣证件</button>
            <button class="btn sm" style="background:#fff0f0;color:#e5484d" @click="openPanel2(i, 'pause')">⏸️ 暂停场次</button>
            <button v-if="isCommittee && i.status === 'resolved'" class="btn green sm" @click="closeCase(i)">📁 结案</button>
          </div>
          <div v-else class="row mt8" style="justify-content:flex-end">
            <button v-if="isCommittee" class="btn ghost sm" @click="reopen(i)">🔓 重新开启</button>
          </div>

          <!-- 操作面板 -->
          <div v-if="panel.id === i.id" class="act-panel">
            <template v-if="panel.mode === 'hold'">
              <div class="ap-title">🪪 暂扣证件并关联本事件</div>
              <select v-model="form2.badge_id" style="width:100%">
                <option value="">选择证件（队伍/运动员/裁判/工作人员）</option>
                <option v-for="b in store.badges" :key="b.id" :value="b.id">{{ b.code }} · {{ b.name }}（{{ SUBJECT_TEXT[b.subject_type] }}）<template v-if="b.status==='blocked'"> — 已停用</template></option>
              </select>
              <input v-model="panel.text" placeholder="暂扣原因（将写入异常审计）" />
            </template>
            <template v-else-if="panel.mode === 'pause'">
              <div class="ap-title">⏸️ 暂停场次</div>
              <select v-model="form2.match_id" style="width:100%">
                <option value="">选择待赛场次（暂停后禁止入场、不可录入比分）</option>
                <option v-for="m in panelMatches" :key="m.id" :value="m.id">{{ matchText(m) }}</option>
              </select>
              <input v-model="panel.text" placeholder="暂停原因（将写入异常审计）" />
            </template>
            <template v-else-if="panel.mode === 'reschedule'">
              <div class="ap-title">🕐 暂停改期：{{ store.matches.find(x => x.id === panel.mid) ? matchText(store.matches.find(x => x.id === panel.mid)) : '' }}</div>
              <div class="row wrap" style="gap:8px">
                <label>新时间 <input v-model="panel.time_label" placeholder="如 16:20" style="width:120px;margin-left:6px" list="incident-time-presets" /></label>
                <datalist id="incident-time-presets"><option v-for="t in ['11:20','12:30','13:00','14:00','14:30','15:00','15:30','16:00','16:20','17:00']" :key="t" :value="t" /></datalist>
                <label>新场地
                  <select v-model="panel.venue_id" style="width:150px;margin-left:6px">
                    <option value="">不更换</option>
                    <option v-for="v in store.venues" :key="v.id" :value="v.id">{{ v.name }}</option>
                  </select>
                </label>
              </div>
              <input v-model="panel.text" placeholder="改期原因（将写入异常审计；执法冲突自动重排）" />
              <div v-if="panel.conflicts" class="conf-box">
                <div v-if="panel.conflicts.venue?.length"><b>🏟️ 场地撞场：</b><div v-for="(c,k) in panel.conflicts.venue" :key="'v'+k" class="conf-line">⛔ {{ c.title }}</div></div>
                <div v-if="panel.conflicts.referee?.length"><b>🧑‍⚖️ 裁判冲突：</b><div v-for="(c,k) in panel.conflicts.referee" :key="'r'+k" class="conf-line">⛔ {{ c.referee }} 同时执法 {{ c.title }}</div></div>
              </div>
              <div v-if="panel.error && !panel.conflicts" class="tag r">{{ panel.error }}</div>
            </template>
            <div class="row" style="justify-content:flex-end;gap:6px">
              <button class="btn ghost sm" @click="closePanel">取消</button>
              <button v-if="panel.mode === 'hold'" class="btn sm btn-danger" @click="confirmHold(i)">确认暂扣</button>
              <button v-else-if="panel.mode === 'pause'" class="btn sm btn-danger" @click="confirmPause(i)">确认暂停</button>
              <template v-else>
                <button v-if="panel.conflicts" class="btn ghost sm" @click="forceReschedule(i)">强制保留冲突并留痕</button>
                <button class="btn primary sm" @click="confirmReschedule(i)">确认改期并恢复</button>
              </template>
            </div>
          </div>

          <!-- 时间线 -->
          <div class="timeline">
            <div v-for="l in i.logs" :key="l.id" class="tl-item">
              <span class="tag" :class="LOG_META[l.action]?.tag">{{ LOG_META[l.action]?.text || l.action }}</span>
              <span class="tl-detail">{{ l.detail }}</span>
              <span class="tl-time">{{ l.actor_name }} · {{ l.created_at }}</span>
            </div>
          </div>
        </div>
        <div v-if="!list.length" class="empty">暂无安全事件</div>
      </div>
    </div>
  </div>
</template>

<style scoped>
.role-switch { width: 150px; font-weight: 700; }
.rules { display: flex; align-items: center; gap: 7px; flex-wrap: wrap; font-size: 12.5px; }
.form-row { display: flex; align-items: center; gap: 10px; }
.form-row label { width: 72px; flex-shrink: 0; font-size: 12px; color: var(--muted); font-weight: 700; }
.form-row select, .form-row input { flex: 1; }
.icard { border: 1px solid var(--line); border-radius: 14px; padding: 14px 16px; background: #fff; box-shadow: var(--shadow-sm); border-left: 4px solid var(--line); }
.icard.reported { border-left-color: var(--accent); }
.icard.triaged, .icard.handling { border-left-color: var(--accent3); }
.icard.resolved { border-left-color: var(--gold); }
.icard.closed { border-left-color: #c3ccd6; background: linear-gradient(180deg, #fafbfc, #fff); }
.iheader { display: flex; justify-content: space-between; align-items: center; gap: 8px; flex-wrap: wrap; font-size: 14px; }
.meta-line { display: flex; gap: 10px; flex-wrap: wrap; align-items: center; font-size: 11.5px; color: var(--muted); margin-top: 8px; }
.detail-line { font-size: 13px; margin-top: 6px; color: var(--ink); line-height: 1.6; }
.triage-box { margin-top: 8px; padding: 8px 12px; background: #f3f8ff; border: 1px solid #d3e6ff; border-radius: 10px; font-size: 12.5px; line-height: 1.7; }
.resolution-box { margin-top: 8px; padding: 8px 12px; background: #f4f6f8; border: 1px solid var(--line); border-radius: 10px; font-size: 12.5px; line-height: 1.7; }
.link-box { margin-top: 10px; display: flex; flex-direction: column; gap: 7px; }
.link-row { display: flex; align-items: center; gap: 9px; flex-wrap: wrap; padding: 8px 11px; background: #fff8f8; border: 1px solid #f6d9da; border-radius: 10px; font-size: 12.5px; }
.act-panel { margin-top: 10px; padding: 12px; background: var(--bg2); border-radius: 12px; display: flex; flex-direction: column; gap: 9px; }
.ap-title { font-weight: 800; font-size: 13px; }
.btn-danger { background: #e5484d; color: #fff; }
.conf-box { background: #fff5f5; border: 1px solid #f0a1a1; border-radius: 10px; padding: 9px 11px; font-size: 12px; }
.conf-line { margin-top: 4px; }
.timeline { margin-top: 10px; padding-top: 9px; border-top: 1px dashed var(--line); display: flex; flex-direction: column; gap: 6px; }
.tl-item { display: flex; align-items: baseline; gap: 8px; font-size: 12px; flex-wrap: wrap; }
.tl-detail { flex: 1; min-width: 220px; color: var(--ink); line-height: 1.6; }
.tl-time { color: var(--muted); font-size: 11px; white-space: nowrap; }
.mini { background: #fff; border: 1px solid var(--line); border-radius: 7px; font-size: 11px; padding: 3px 8px; color: var(--muted); font-weight: 600; }
.mini:hover { border-color: #e5484d; color: #e5484d; }
.mini.ok:hover { border-color: var(--accent2); color: var(--accent2); }
.slide-enter-active, .slide-leave-active { transition: all .22s ease; }
.slide-enter-from, .slide-leave-to { opacity: 0; transform: translateY(-8px); }
</style>
