<script setup>
import { ref, computed, reactive } from 'vue'
import { useEventStore } from '@/store/event'
const store = useEventStore()

const toast = ref('')
const showToast = msg => { toast.value = msg; setTimeout(() => toast.value = '', 4000) }

/* ---------- 常量 ---------- */
const TYPE_META = {
  match: { icon: '⚽', text: '比分 / 赛果' },
  track: { icon: '⏱️', text: '田径成绩' },
  eligibility: { icon: '🪪', text: '参赛资格' }
}
const STATUS_META = {
  pending: { tag: 'o', text: '待复核' },
  upheld: { tag: 'g', text: '复核成立' },
  rejected: { tag: 'r', text: '已驳回' },
  withdrawn: { tag: 'gray', text: '已撤申' }
}
const ACTION_META = {
  submit: '提交异议', withdraw: '单位撤申', reject: '驳回维持',
  amend_match: '改判比分', amend_track: '更正成绩',
  revoke_eligibility: '撤销资格', restore_eligibility: '恢复资格'
}

/* ---------- 统计卡 ---------- */
const stats = computed(() => ({
  total: store.appeals.length,
  pending: store.appeals.filter(a => a.status === 'pending').length,
  upheld: store.appeals.filter(a => a.status === 'upheld').length,
  rejected: store.appeals.filter(a => a.status === 'rejected').length
}))

/* ---------- 提交申诉 ---------- */
const form = reactive({ target_type: 'match', target_kind: 'team', target_id: null, unit_id: null, reason: '', evidence: '' })
function switchType(t) {
  form.target_type = t
  form.target_id = null
}
// 可申诉对象：已完赛球类场次 / 已结算田径成绩 / 已通过或已撤销的队伍运动员
const matchOptions = computed(() => store.matches
  .filter(m => m.status === 'finished' && m.team_a && m.team_b && store.sports.find(s => s.id === m.sport_id)?.format !== 'track')
  .map(m => ({ id: m.id, label: `${store.sports.find(s => s.id === m.sport_id)?.name} · ${m.stage}${m.group_name ? ' · ' + m.group_name : ''}｜${m.teamA?.name} ${m.score_a}:${m.score_b}${m.tb_a != null ? `（决胜${m.tb_a}:${m.tb_b}）` : ''} ${m.teamB?.name}` })))
const trackOptions = computed(() => store.entries
  .filter(e => e.mark != null && store.athletes.find(a => a.id === e.athlete_id)?.status === 'approved')
  .map(e => ({ id: e.athlete_id, label: `${store.sports.find(s => s.id === e.sport_id)?.name}｜${e.aname}（${e.unit}）现成绩 ${e.mark}s · 第${e.rank}名` })))
const eligibilityTeams = computed(() => store.teams
  .filter(t => ['approved', 'revoked'].includes(t.status))
  .map(t => ({ id: t.id, label: `队伍｜${t.name}（${t.unit} · ${store.sports.find(s => s.id === t.sport_id)?.name}）· ${t.status === 'approved' ? '已通过' : '已撤销'}` })))
const eligibilityAthletes = computed(() => store.athletes
  .filter(a => ['approved', 'revoked'].includes(a.status))
  .map(a => ({ id: a.id, label: `运动员｜${a.name}（${a.unit} · ${store.sports.find(s => s.id === a.sport_id)?.name}）· ${a.status === 'approved' ? '已通过' : '已撤销'}` })))
const eligibilityOptions = computed(() => form.target_kind === 'team' ? eligibilityTeams.value : eligibilityAthletes.value)
const targetOptions = computed(() =>
  form.target_type === 'match' ? matchOptions.value :
  form.target_type === 'track' ? trackOptions.value : eligibilityOptions.value)

async function submit() {
  if (!form.unit_id) return showToast('⚠️ 请选择提起异议的参赛单位')
  if (!form.target_id) return showToast('⚠️ 请选择申诉对象')
  if (!form.reason.trim()) return showToast('⚠️ 异议内容不能为空')
  try {
    const r = await store.submitAppeal({
      target_type: form.target_type,
      target_kind: form.target_type === 'eligibility' ? form.target_kind : null,
      target_id: Number(form.target_id), unit_id: Number(form.unit_id),
      reason: form.reason.trim(), evidence: form.evidence.trim()
    })
    showToast(`✅ 异议已提交（${r.code}），等待仲裁委员会复核`)
    form.target_id = null; form.reason = ''; form.evidence = ''
  } catch (e) { showToast('⚠️ ' + e.message) }
}

/* ---------- 列表与筛选 ---------- */
const statusFilter = ref('all')
const list = computed(() => store.appeals.filter(a => statusFilter.value === 'all' || a.status === statusFilter.value))

/* ---------- 撤申 ---------- */
async function withdraw(a) {
  try {
    await store.withdrawAppeal(a.id, { operator: a.unit_name + '领队', note: '单位主动撤申' })
    showToast('✅ 申诉已撤回并留痕')
  } catch (e) { showToast('⚠️ ' + e.message) }
}

/* ---------- 复核面板 ---------- */
const acting = ref(null)
const review = reactive({ note: '', score_a: null, score_b: null, tb_a: null, tb_b: null, mark: null })
function openReview(a) {
  acting.value = a
  review.note = ''
  if (a.target_type === 'match' && a.target) {
    review.score_a = a.target.score_a; review.score_b = a.target.score_b
    review.tb_a = a.target.tb_a; review.tb_b = a.target.tb_b
  }
  if (a.target_type === 'track' && a.target) review.mark = a.target.mark
}
function closeReview() { acting.value = null }

const selectedMatch = computed(() => acting.value?.target_type === 'match' ? acting.value.target : null)
const needTB = computed(() => selectedMatch.value && ['半决赛', '决赛', '季军'].includes(selectedMatch.value.stage)
  && Number(review.score_a) === Number(review.score_b))

async function submitReview(verdict) {
  if (!review.note.trim()) return showToast('⚠️ 必须填写复核意见（全量留痕）')
  if (verdict === 'uphold' && acting.value.target_type === 'match') {
    const sa = Number(review.score_a), sb = Number(review.score_b)
    if (!Number.isInteger(sa) || !Number.isInteger(sb) || sa < 0 || sb < 0) return showToast('⚠️ 改判比分必须为非负整数')
    if (needTB.value) {
      const ta = Number(review.tb_a), tb = Number(review.tb_b)
      if (!Number.isInteger(ta) || !Number.isInteger(tb) || ta < 0 || tb < 0 || ta === tb) {
        return showToast('⚠️ 淘汰赛平分需录入决胜比分且不能再次持平')
      }
    }
  }
  if (verdict === 'uphold' && acting.value.target_type === 'track') {
    const mark = Number(review.mark)
    if (!Number.isFinite(mark) || mark <= 0) return showToast('⚠️ 更正成绩必须为大于 0 的秒数')
  }
  try {
    const r = await store.reviewAppeal(acting.value.id, {
      verdict,
      review_note: review.note.trim(),
      score_a: review.score_a, score_b: review.score_b,
      tb_a: needTB.value ? review.tb_a : null, tb_b: needTB.value ? review.tb_b : null,
      mark: review.mark
    })
    if (r.idempotent) { showToast('ℹ️ 该申诉已复核，返回首次结论（幂等）'); closeReview(); return }
    if (verdict === 'reject') showToast('✅ 申诉已驳回，维持原赛果 / 原资格')
    else {
      const c = r.correction
      const parts = []
      if (c.impact) {
        if (c.impact.downstream_voided) parts.push(`取消后续 ${c.impact.downstream_voided} 场`)
        if (c.impact.downstream_created) parts.push(`递补生成 ${c.impact.downstream_created} 场`)
        if (c.impact.walkover) parts.push(`弃权 ${c.impact.walkover} 场`)
        if (c.impact.voided) parts.push(`取消成绩 ${c.impact.voided} 场`)
        if ((c.impact.replacements || 0) + (c.impact.cascade || 0)) parts.push(`淘汰赛递补/级联 ${(c.impact.replacements || 0) + (c.impact.cascade || 0)} 场`)
        if (c.impact.walkover_reverted) parts.push(`弃权回退 ${c.impact.walkover_reverted} 场`)
        if (c.impact.voided_reverted) parts.push(`重赛 ${c.impact.voided_reverted} 场`)
        if (c.impact.entries || c.impact.entries_restored) parts.push(`成绩档案 ${c.impact.entries || c.impact.entries_restored} 条`)
      }
      if (c.rank_changes) parts.push(`名次连带变动 ${c.rank_changes.length} 人`)
      parts.push(`奖牌变动 ${(c.medal_changes || []).length} 个单位`)
      if (c.standings_changes) parts.push(`积分排名变动 ${c.standings_changes.length} 队`)
      showToast('✅ 复核成立已回写：' + parts.join(' · '))
    }
    closeReview()
  } catch (e) { showToast('⚠️ ' + e.message) }
}

/* ---------- 目标状态徽章（资格类给出撤销/恢复两个结论入口） ---------- */
function eligibilityVerdict(a) {
  return a.target?.status === 'revoked' ? 'restore' : 'revoke'
}

/* ---------- 奖牌变动展示 ---------- */
const fmtDelta = n => (n > 0 ? '+' + n : String(n))
const deltaColor = n => n > 0 ? 'var(--accent2)' : '#e5484d'
const rankText = x => `${x.before_rank ?? '—'} → ${x.after_rank ?? '—'}`

const unitColor = uid => store.unitOfUid(uid)?.color || '#ccc'
</script>

<template>
  <div v-if="store.loaded">
    <div class="page-h">
      <div><h2>⚖️ 赛事申诉复核</h2><div class="sub">参赛单位提交异议 · 仲裁委员会复核 · 回写比分/成绩/资格并联动弃权、淘汰赛递补、积分与奖牌全量重算</div></div>
      <div v-if="toast" class="toast">{{ toast }}</div>
    </div>

    <!-- 统计卡 -->
    <div class="grid" style="grid-template-columns:repeat(4,1fr)">
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#ff7a2f,#ffb27e)"></span><span class="ic">📨</span><b>{{ stats.total }}</b><em>申诉单总数</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#ffb92b,#ffd98a)"></span><span class="ic">⏳</span><b>{{ stats.pending }}</b><em>待复核</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#22c15e,#7edda4)"></span><span class="ic">✅</span><b>{{ stats.upheld }}</b><em>复核成立（已改判/处置）</em></div>
      <div class="card stat"><span class="bar" style="background:linear-gradient(90deg,#9aa7b5,#c2cad4)"></span><span class="ic">🚫</span><b>{{ stats.rejected }}</b><em>驳回 / 撤申</em></div>
    </div>

    <div class="grid g2 mt">
      <!-- 提交异议 -->
      <div class="card">
        <div class="caption">📝 提交异议 <span class="hint">参赛单位对赛果 / 成绩 / 资格提起申诉</span></div>
        <div class="pad">
          <div class="form-row">
            <label>申诉单位</label>
            <select v-model.number="form.unit_id">
              <option :value="null" disabled>选择参赛单位</option>
              <option v-for="u in store.units" :key="u.id" :value="u.id">{{ u.name }}</option>
            </select>
          </div>
          <div class="form-row">
            <label>申诉类型</label>
            <div class="filters">
              <button v-for="(meta, key) in TYPE_META" :key="key" class="chip" :class="{ on: form.target_type === key }" @click="switchType(key)">{{ meta.icon }} {{ meta.text }}</button>
              <template v-if="form.target_type === 'eligibility'">
                <button class="chip" :class="{ on: form.target_kind === 'team' }" @click="form.target_kind = 'team'; form.target_id = null">队伍</button>
                <button class="chip" :class="{ on: form.target_kind === 'athlete' }" @click="form.target_kind = 'athlete'; form.target_id = null">运动员</button>
              </template>
            </div>
          </div>
          <div class="form-row">
            <label>申诉对象</label>
            <select v-model.number="form.target_id">
              <option :value="null" disabled>
                {{ form.target_type === 'match' ? '选择已完赛场次' : form.target_type === 'track' ? '选择已结算成绩的运动员' : '选择队伍 / 运动员' }}
              </option>
              <option v-for="o in targetOptions" :key="o.id" :value="o.id">{{ o.label }}</option>
            </select>
          </div>
          <div class="form-row align-top">
            <label>异议内容</label>
            <textarea v-model="form.reason" rows="3" placeholder="陈述异议事实与诉求（必填）"></textarea>
          </div>
          <div class="form-row align-top">
            <label>证据材料</label>
            <textarea v-model="form.evidence" rows="2" placeholder="录像 / 技术统计表 / 注册档案 / 证人等（选填，随单留痕）"></textarea>
          </div>
          <div class="row" style="justify-content:flex-end"><button class="btn primary" @click="submit">📨 提交申诉单</button></div>
        </div>
      </div>

      <!-- 复核规则 -->
      <div class="card">
        <div class="caption">ℹ️ 复核规则与联动处置</div>
        <div class="pad" style="font-size:13px;line-height:1.95;color:var(--ink)">
          <p>· <b>比分 / 赛果申诉</b>：成立后按复核比分回写，半决赛改判将<b>取消原决赛/季军战并按新赛果递补重赛</b>；淘汰赛平分须同步录入决胜比分。</p>
          <p>· <b>田径成绩申诉</b>：成立后回写计时成绩，全部名次<b>连带重排</b>，金/银/铜重算。</p>
          <p>· <b>参赛资格申诉</b>：成立可<b>撤销资格</b>（未赛判弃权 3:0、已赛取消成绩、淘汰赛按同组名次递补）；已撤销对象也可经申诉<b>恢复资格</b>（弃权回待赛、取消场次安排重赛、淘汰赛重新递补）。</p>
          <p>· 任一复核结论必须填写<b>复核意见</b>；提交、撤申、驳回、改判、资格撤销/恢复与每一笔级联处置均<b>全量审计留痕</b>。</p>
          <p>· 复核为<b>幂等操作</b>：双击 / 重试 / 并发提交只生效一次；同一对象待复核申诉唯一。</p>
        </div>
      </div>
    </div>

    <!-- 申诉列表 -->
    <div class="card mt">
      <div class="caption">
        <span>🗂️ 申诉工单 <span class="hint">共 {{ list.length }} 条</span></span>
        <div class="filters">
          <button class="chip" :class="{ on: statusFilter === 'all' }" @click="statusFilter = 'all'">全部</button>
          <button v-for="(meta, key) in STATUS_META" :key="key" class="chip" :class="{ on: statusFilter === key }" @click="statusFilter = key">{{ meta.text }}</button>
        </div>
      </div>
      <div class="pad" style="display:flex;flex-direction:column;gap:12px">
        <div v-for="a in list" :key="a.id" class="acard">
          <div class="mheader">
            <span>
              <b class="mono">{{ a.code }}</b>
              <span class="tag b" style="margin-left:8px">{{ TYPE_META[a.target_type].icon }} {{ TYPE_META[a.target_type].text }}</span>
              <span class="tag gray" style="margin-left:4px">{{ a.sport_name }}</span>
            </span>
            <span class="tag" :class="STATUS_META[a.status].tag">{{ STATUS_META[a.status].text }}</span>
          </div>

          <div class="mrow" style="margin-top:8px">
            <span class="badge"><span class="dot" :style="{ background: a.unit_color || '#ccc' }"></span>{{ a.unit_name }}</span>
            <span class="hint" style="font-size:11px">提交于 {{ a.submitted_at }}<span v-if="a.reviewer"> · 复核人 {{ a.reviewer }} · {{ a.reviewed_at }}</span></span>
          </div>
          <div class="target-line">🎯 {{ a.target?.title || '（对象已不存在）' }}</div>
          <div class="reason-line">💬 {{ a.reason }}</div>
          <div v-if="a.evidence" class="reason-line muted">📎 证据：{{ a.evidence }}</div>
          <div v-if="a.review_note" class="verdict-line">⚖️ 复核意见：{{ a.review_note }}</div>

          <!-- 改判快照 -->
          <template v-if="a.correction">
            <div v-if="a.correction.kind === 'match'" class="snap">
              <div class="snap-title">📼 赛果回写</div>
              <div>原 <b class="mono">{{ a.correction.before.score_text }}</b>（{{ a.correction.before.winner || '平局' }}）
                ➜ 新 <b class="mono" style="color:var(--accent)">{{ a.correction.after.score_text }}</b>（{{ a.correction.after.winner || '平局' }}）</div>
              <div v-if="a.correction.impact.downstream_voided || a.correction.impact.downstream_created" class="hint">
                淘汰赛联动：取消后续场次 {{ a.correction.impact.downstream_voided }} 场 · 按新赛果递补生成 {{ a.correction.impact.downstream_created }} 场
              </div>
            </div>
            <div v-else-if="a.correction.kind === 'track'" class="snap">
              <div class="snap-title">📼 成绩回写</div>
              <div>{{ a.correction.athlete }}：<b class="mono">{{ a.correction.before.mark }}s（第{{ a.correction.before.rank }}名）</b>
                ➜ <b class="mono" style="color:var(--accent)">{{ a.correction.after.mark }}s（第{{ a.correction.after.rank }}名）</b></div>
              <div v-if="a.correction.rank_changes?.length" class="rank-chips">
                <span v-for="r in a.correction.rank_changes" :key="r.athlete" class="tag gray">{{ r.athlete }} {{ rankText(r) }}</span>
              </div>
            </div>
            <div v-else class="snap">
              <div class="snap-title">📼 资格处置（{{ a.correction.action === 'revoke_eligibility' ? '撤销资格' : '恢复资格' }}）</div>
              <div class="hint">{{ a.correction.target_kind === 'team' ? '队伍' : '运动员' }}「{{ a.correction.target }}」·
                弃权 {{ a.correction.impact.walkover ?? 0 }} · 取消成绩 {{ a.correction.impact.voided ?? 0 }} ·
                弃权回退 {{ a.correction.impact.walkover_reverted ?? 0 }} · 重赛 {{ a.correction.impact.voided_reverted ?? 0 }} ·
                递补级联 {{ (a.correction.impact.replacements ?? 0) + (a.correction.impact.cascade ?? 0) }} ·
                成绩档案 {{ a.correction.impact.entries ?? a.correction.impact.entries_restored ?? 0 }}
              </div>
            </div>

            <!-- 奖牌变动 -->
            <div v-if="a.correction.medal_changes?.length" class="medal-delta">
              <div class="snap-title">🥇 奖牌回写</div>
              <span v-for="(m, i) in a.correction.medal_changes" :key="i" class="md-chip">
                {{ m.unit }}：
                <b :style="{ color: deltaColor(m.gold) }">金 {{ fmtDelta(m.gold) }}</b>
                <b :style="{ color: deltaColor(m.silver) }">银 {{ fmtDelta(m.silver) }}</b>
                <b :style="{ color: deltaColor(m.bronze) }">铜 {{ fmtDelta(m.bronze) }}</b>
              </span>
            </div>
            <!-- 积分排名变动 -->
            <div v-if="a.correction.standings_changes?.length" class="medal-delta">
              <div class="snap-title">🧮 积分榜变动</div>
              <span v-for="(s, i) in a.correction.standings_changes" :key="i" class="tag gray" style="margin:2px 6px 2px 0">
                {{ s.team }} 名次 {{ rankText(s) }} · 积分 {{ s.before_points ?? '—' }} → {{ s.after_points ?? '—' }}
              </span>
            </div>
          </template>

          <!-- 操作区 -->
          <div class="row mt8" style="justify-content:flex-end;gap:6px;flex-wrap:wrap">
            <template v-if="a.status === 'pending'">
              <button class="btn ghost sm" @click="withdraw(a)">↩️ 单位撤申</button>
              <button class="btn sm" style="background:#ffecec;color:#e5484d" @click="openReview(a)">🚫 驳回维持</button>
              <button v-if="a.target_type === 'eligibility'" class="btn primary sm" @click="openReview(a)">
                {{ eligibilityVerdict(a) === 'revoke' ? '⚔ 复核成立 · 撤销资格' : '♻️ 复核成立 · 恢复资格' }}
              </button>
              <button v-else class="btn primary sm" @click="openReview(a)">✅ 复核成立 · 回写改判</button>
            </template>
            <span v-else class="hint">{{ a.status === 'withdrawn' ? '申诉单位已撤申' : '已出具复核结论并归档' }}</span>
          </div>

          <!-- 复核面板 -->
          <div v-if="acting?.id === a.id" class="review-panel">
            <!-- 比分改判 -->
            <template v-if="a.target_type === 'match'">
              <div class="rp-title">改判比分（原记录 {{ a.target?.score_a }}:{{ a.target?.score_b }}<template v-if="a.target?.tb_a != null">；决胜 {{ a.target.tb_a }}:{{ a.target.tb_b }}</template>）</div>
              <div class="row" style="gap:8px;flex-wrap:wrap">
                <span>{{ a.target?.teamA?.name }}</span>
                <input v-model.number="review.score_a" type="number" min="0" class="score-in" style="width:64px"> :
                <input v-model.number="review.score_b" type="number" min="0" class="score-in" style="width:64px">
                <span>{{ a.target?.teamB?.name }}</span>
              </div>
              <div v-if="needTB" class="tbrow">
                <span>⚔️ 淘汰赛平分 · 加时/点球决胜：</span>
                <input v-model.number="review.tb_a" type="number" min="0" class="score-in" style="width:56px"><b>:</b>
                <input v-model.number="review.tb_b" type="number" min="0" class="score-in" style="width:56px">
              </div>
            </template>
            <!-- 田径更正 -->
            <template v-else-if="a.target_type === 'track'">
              <div class="rp-title">更正计时成绩（原成绩 {{ a.target?.mark }}s · 第{{ a.target?.mark_rank }}名）</div>
              <div class="row"><span>{{ a.target?.aname }}</span><input v-model.number="review.mark" type="number" step="0.01" min="0.01" style="width:110px"><span>秒（越小越优，全体名次连带重排）</span></div>
            </template>
            <!-- 资格 -->
            <template v-else>
              <div class="rp-title">{{ eligibilityVerdict(a) === 'revoke' ? '撤销资格将级联：未赛判弃权 3:0、已赛取消成绩、淘汰赛同组递补、积分奖牌重算' : '恢复资格将级联：弃权场次回待赛、取消场次重赛、淘汰赛重新递补、积分奖牌重算' }}</div>
            </template>
            <input v-model="review.note" placeholder="复核意见（必填，写入全量审计）" style="width:100%;margin-top:10px">
            <div class="row mt8" style="justify-content:flex-end;gap:6px">
              <button class="btn ghost sm" @click="closeReview">取消</button>
              <button class="btn sm" style="background:#ffecec;color:#e5484d" @click="submitReview('reject')">🚫 驳回</button>
              <button v-if="a.target_type === 'eligibility'" class="btn green sm" @click="submitReview(eligibilityVerdict(a))">
                {{ eligibilityVerdict(a) === 'revoke' ? '确认撤销资格' : '确认恢复资格' }}
              </button>
              <button v-else class="btn green sm" @click="submitReview('uphold')">✅ 确认改判并联动重算</button>
            </div>
          </div>
        </div>
        <div v-if="!list.length" class="empty">暂无该状态的申诉单</div>
      </div>
    </div>

    <!-- 全量审计 -->
    <div class="card mt">
      <div class="caption">🛡️ 申诉复核全量审计 <span class="hint">提交 / 撤申 / 驳回 / 改判 / 成绩更正 / 资格撤销 / 恢复，逐笔可追溯（共 {{ store.appealLogs.length }} 条）</span></div>
      <div class="pad" style="overflow-x:auto">
        <table>
          <thead><tr><th>时间</th><th>动作</th><th>单位</th><th>项目</th><th>变更快照</th><th>原因 / 意见</th><th>操作人</th></tr></thead>
          <tbody>
            <tr v-for="l in store.appealLogs" :key="l.id">
              <td class="mono" style="white-space:nowrap">{{ l.created_at }}</td>
              <td><span class="tag" :class="l.action === 'submit' || l.action === 'withdraw' ? 'b' : l.action === 'reject' ? 'r' : 'g'">{{ ACTION_META[l.action] || l.action }}</span></td>
              <td>{{ l.unit_name || '—' }}</td>
              <td>{{ l.sport_name || '—' }}</td>
              <td class="ph" style="max-width:360px">{{ l.detail }}</td>
              <td class="ph" style="max-width:200px">{{ l.reason || '—' }}</td>
              <td>{{ l.operator }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<style scoped>
.form-row { display: flex; align-items: center; gap: 10px; margin-bottom: 12px; }
.form-row.align-top { align-items: flex-start; }
.form-row label { width: 78px; flex-shrink: 0; font-size: 12px; color: var(--muted); font-weight: 700; padding-top: 8px; }
.form-row select, .form-row textarea { flex: 1; width: 100%; }
.acard { border: 1px solid var(--line); border-radius: 14px; padding: 13px 15px; background: #fff; box-shadow: var(--shadow-sm); }
.target-line { font-size: 13px; font-weight: 700; margin-top: 8px; }
.reason-line { font-size: 12.5px; margin-top: 6px; padding: 6px 10px; background: #f8fafc; border-radius: 8px; border-left: 3px solid var(--accent3); }
.reason-line.muted { border-left-color: var(--muted); color: var(--muted); }
.verdict-line { font-size: 12.5px; margin-top: 6px; padding: 6px 10px; background: #fff7ed; border-radius: 8px; border-left: 3px solid var(--accent); font-weight: 600; }
.snap, .medal-delta { margin-top: 8px; padding: 9px 12px; background: #f3faf5; border: 1px dashed #b9e6c4; border-radius: 10px; font-size: 12.5px; line-height: 1.8; }
.medal-delta { background: #fffaf0; border-color: #f0d29a; }
.snap-title, .medal-delta .snap-title { font-weight: 800; font-size: 12px; color: var(--muted); margin-bottom: 3px; }
.rank-chips { margin-top: 4px; }
.md-chip { display: inline-block; margin: 2px 12px 2px 0; font-size: 12px; }
.md-chip b { margin-left: 4px; font-family: Consolas, monospace; }
.review-panel { margin-top: 10px; padding: 12px; background: var(--bg2); border-radius: 12px; }
.rp-title { font-size: 12px; font-weight: 700; color: var(--muted); margin-bottom: 8px; }
.score-in { font-weight: 800; font-size: 15px; text-align: center; }
.tbrow { display: flex; align-items: center; gap: 6px; margin-top: 8px; padding: 7px 10px; border-radius: 10px; background: #fff7ed; border: 1px dashed var(--accent); font-size: 12px; color: var(--muted); flex-wrap: wrap; }
</style>
