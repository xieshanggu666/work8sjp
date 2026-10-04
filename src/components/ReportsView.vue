<script setup>
import { computed } from 'vue'
import { useEventStore } from '@/store/event'
const store = useEventStore()

const bestAttack = computed(() => {
  const rows = []
  store.sports.forEach(s => {
    (store.standings[s.id] || []).forEach(r => { if (r.play > 0) rows.push({ t: r.tname, sport: s.name, gf: r.gf - r.ga, unit: r.unit, color: r.color }) })
  })
  return rows.sort((a, b) => b.gf - a.gf).slice(0, 6)
})
const maxGD = computed(() => Math.max(1, ...bestAttack.value.map(r => r.gf)))

const unitPoints = computed(() => {
  const m = {}
  const u = store.units
  u.forEach(x => m[x.id] = { name: x.name, color: x.color, pts: 0, gold: 0 })
  store.teams.forEach(t => {
    const st = (store.standings[t.sport_id] || []).find(r => r.team_id === t.id)
    if (st && m[t.unit_id]) m[t.unit_id].pts += st.points
  })
  store.medals.forEach(md => { if (m[md.unit_id]) m[md.unit_id].gold = md.gold })
  return Object.values(m).sort((a, b) => b.pts - a.pts)
})
const maxPts = computed(() => Math.max(1, ...unitPoints.value.map(x => x.pts)))

// —— 裁判排班联动报表 ——
const refereeRows = computed(() => store.workload)
const maxDone = computed(() => Math.max(1, ...refereeRows.value.map(r => (r.done || 0) + (r.upcoming || 0))))
// 整场覆盖率：按执法席位（主裁/助理/记录台）统计
const coverage = computed(() => store.conflicts?.coverage?.overall || { slots_need: 0, slots_filled: 0, slots_pct: 100, roles: {} })
const upcomingCoverage = computed(() => store.conflicts?.coverage?.scheduled || { slots_need: 0, slots_filled: 0, slots_pct: 100, roles: {} })
const finishedCoverage = computed(() => store.conflicts?.coverage?.finished || { slots_need: 0, slots_filled: 0, slots_pct: 100, roles: {} })
const roleLabels = { chief: '主裁', assistant: '助理裁判', recorder: '记录台' }
const recentLogs = computed(() => store.assignmentLogs.slice(0, 6)
  .map(l => ({ ...l, name: { assign: '排班', force_assign: '强制排班', auto_assign: '自动排班', release: '解除', reassign: '临时调班', swap: '调班对调', match_change: '赛程变更', reschedule_rollback: '改期回滚', schedule_added: '赛程新增', schedule_rebuild: '赛程重排', match_finish: '完赛归档', void_release: '取消解除' }[l.action] || l.action })))

// —— 申诉复核审计 ——
const appealStat = computed(() => {
  const c = { pending: 0, reviewing: 0, upheld: 0, rejected: 0, withdrawn: 0 }
  store.appeals.forEach(a => { c[a.status] = (c[a.status] || 0) + 1 })
  return c
})
function appealBrief(a) {
  const t = a.target
  if (!t) return '（对象已不存在）'
  if (a.target_type === 'match') return `${t.teamA?.name} ${t.score_a}:${t.score_b} ${t.teamB?.name}`
  if (a.target_type === 'track') return `${t.aname} ${t.mark}s（第${t.rank}名）`
  return `「${t.name}」资格（${t.target_unit}）`
}
function appealImpact(a) {
  if (a.status !== 'upheld' || !a.impact) return '—'
  const im = a.impact, p = []
  if (im.resolution === 'revoked') p.push(`撤销资格·弃权${im.walkover || 0}·取消${im.voided || 0}场`)
  else if (im.resolution === 'track_corrected') p.push('成绩回写并重排名')
  else p.push('比分回写')
  if ((im.replays || []).length) p.push(`重赛/递补${im.replays.length}场`)
  else if (im.replacements) p.push(`递补${im.replacements}场`)
  if (im.medal_changes?.length) p.push(`${im.medal_changes.length}单位奖牌变动`)
  return p.join('；')
}
</script>

<template>
  <div v-if="store.loaded">
    <div class="page-h">
      <div><h2>📊 报表中心</h2><div class="sub">赛事数据洞察：净胜球排行 · 综合积分 · 项目概览</div></div>
    </div>

    <div class="grid g2">
      <!-- 净胜球排行 -->
      <div class="card">
        <div class="caption">🔥 最佳攻击线（净胜球）</div>
        <div class="pad" style="display:flex;flex-direction:column;gap:14px">
          <div v-for="(r, ri) in bestAttack" :key="ri + '-' + r.t">
            <div class="row spread" style="margin-bottom:6px">
              <span class="badge"><span class="dot" :style="{ background: r.color }"></span>{{ r.t }} <span class="tag gray" style="margin-left:4px">{{ r.sport }}</span></span>
              <b class="mono">{{ r.gf > 0 ? '+' : '' }}{{ r.gf }}</b>
            </div>
            <div class="hbar"><i :style="{ width: (r.gf / maxGD) * 100 + '%', background: r.gf > 0 ? 'var(--accent)' : '#e5484d' }"></i></div>
          </div>
        </div>
      </div>

      <!-- 单位综合积分 -->
      <div class="card">
        <div class="caption">🏅 单位综合积分（球类积分之和）</div>
        <div class="pad" style="display:flex;flex-direction:column;gap:14px">
          <div v-for="u in unitPoints" :key="u.name">
            <div class="row spread" style="margin-bottom:6px">
              <span class="badge"><span class="dot" :style="{ background: u.color }"></span>{{ u.name }}</span>
              <span><span class="tag y">🥇{{ u.gold }}</span> <b class="mono" style="font-size:16px;color:var(--accent)">{{ u.pts }}</b></span>
            </div>
            <div class="hbar"><i :style="{ width: (u.pts / maxPts) * 100 + '%' }"></i></div>
          </div>
        </div>
      </div>
    </div>

    <!-- 裁判执法工作量与排班覆盖 -->
    <div class="grid g2 mt">
      <div class="card">
        <div class="caption">🧑‍⚖️ 裁判执法工作量 <span class="hint">已完赛 / 待赛（主裁·助理·记录台）</span></div>
        <div class="pad" style="display:flex;flex-direction:column;gap:13px">
          <div v-for="r in refereeRows" :key="r.id">
            <div class="row spread" style="margin-bottom:6px">
              <span class="badge">{{ r.name }} <span class="tag gray" style="margin-left:4px">{{ r.sport || '综合执法' }} · {{ r.level }}</span></span>
              <b class="mono"><span style="color:var(--accent2)">{{ r.done || 0 }}</span> / <span style="color:var(--accent3)">{{ r.upcoming || 0 }}</span></b>
            </div>
            <div class="hbar-duo">
              <i class="d" :style="{ width: ((r.done || 0) / maxDone) * 100 + '%' }"></i><i class="u" :style="{ width: ((r.upcoming || 0) / maxDone) * 100 + '%' }"></i>
            </div>
            <div class="ph" style="font-size:11px;margin-top:3px">已执法 主{{ r.done_chief || 0 }}·助{{ r.done_assistant || 0 }}·台{{ r.done_recorder || 0 }}　待执法 主{{ r.upcoming_chief || 0 }}·助{{ r.upcoming_assistant || 0 }}·台{{ r.upcoming_recorder || 0 }}</div>
          </div>
        </div>
      </div>
      <div class="card">
        <div class="caption">📡 整场排班覆盖率与最新变更 <span class="hint">已完赛归档 + 待赛排班</span></div>
        <div class="pad">
          <div class="row spread" style="margin-bottom:8px">
            <span class="badge">执法席位覆盖率（主裁/助理/记录台）</span>
            <b class="mono" style="font-size:16px;color:var(--accent)">{{ coverage.slots_filled }}/{{ coverage.slots_need }}（{{ coverage.slots_pct }}%）</b>
          </div>
          <div class="hbar"><i :style="{ width: coverage.slots_pct + '%', background: coverage.slots_pct === 100 ? 'var(--accent2)' : 'var(--accent)' }"></i></div>
          <div class="grid g2 mt8" style="gap:8px">
            <div class="role-mini">
              <div class="row spread"><span class="ph" style="font-size:12px">已完赛归档</span><b class="mono">{{ finishedCoverage.slots_filled }}/{{ finishedCoverage.slots_need }}（{{ finishedCoverage.slots_pct }}%）</b></div>
              <div class="hbar"><i :style="{ width: finishedCoverage.slots_pct + '%', background: 'var(--accent2)' }"></i></div>
            </div>
            <div class="role-mini">
              <div class="row spread"><span class="ph" style="font-size:12px">待赛排班</span><b class="mono">{{ upcomingCoverage.slots_filled }}/{{ upcomingCoverage.slots_need }}（{{ upcomingCoverage.slots_pct }}%）</b></div>
              <div class="hbar"><i :style="{ width: upcomingCoverage.slots_pct + '%', background: 'var(--accent3)' }"></i></div>
            </div>
          </div>
          <div class="grid g3 mt16" style="gap:10px">
            <div v-for="role in ['chief','assistant','recorder']" :key="role" class="role-mini">
              <div class="row spread"><span class="ph" style="font-size:12px">{{ roleLabels[role] }}</span><b class="mono">{{ coverage.roles[role]?.filled || 0 }}/{{ coverage.roles[role]?.need || 0 }}</b></div>
              <div class="hbar"><i :style="{ width: (coverage.roles[role]?.pct || 100) + '%', background: (coverage.roles[role]?.pct || 100) === 100 ? 'var(--accent2)' : 'var(--accent3)' }"></i></div>
            </div>
          </div>
          <div v-if="store.conflicts?.crew_gaps?.length" class="tag o mt16">🟠 {{ store.conflicts.crew_gaps.length }} 场执法名单不齐</div>
          <div class="row spread mt16" style="margin-bottom:8px"><span class="badge">⛔ 未决冲突</span>
            <b :style="{ color: (store.conflicts?.referee_conflicts.length || 0) + (store.conflicts?.venue_conflicts?.filter(c => c.operational).length || 0) ? '#e5484d' : 'var(--accent2)' }">
              {{ (store.conflicts?.referee_conflicts.length || 0) + (store.conflicts?.venue_conflicts?.filter(c => c.operational).length || 0) }} 起
            </b>
          </div>
          <table style="margin-top:6px">
            <thead><tr><th>操作</th><th>详情</th><th>操作人</th></tr></thead>
            <tbody>
              <tr v-for="l in recentLogs" :key="l.id">
                <td style="white-space:nowrap">{{ l.name }}</td>
                <td class="ph" style="max-width:280px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">{{ l.detail }}</td>
                <td>{{ l.operator }}</td>
              </tr>
            </tbody>
          </table>
        </div>
      </div>
    </div>

    <!-- 申诉复核与改判审计 -->
    <div class="card mt">
      <div class="caption">⚖️ 赛事申诉复核与改判审计 <span class="hint">单位异议 → 受理 → 改判/驳回，比分·积分·奖牌全程留痕</span></div>
      <div class="pad">
        <div class="row" style="gap:10px;flex-wrap:wrap;margin-bottom:14px">
          <span class="tag o">待受理 {{ appealStat.pending }}</span>
          <span class="tag b">复核中 {{ appealStat.reviewing }}</span>
          <span class="tag g">已改判 {{ appealStat.upheld }}</span>
          <span class="tag r">已驳回 {{ appealStat.rejected }}</span>
          <span class="tag gray">已撤案 {{ appealStat.withdrawn }}</span>
        </div>
        <table>
          <thead><tr><th>编号</th><th>单位</th><th>类型</th><th>申诉对象</th><th>状态</th><th>改判/复核意见</th><th>影响摘要</th><th>经办</th></tr></thead>
          <tbody>
            <tr v-for="a in store.appeals.slice(0, 12)" :key="a.id">
              <td class="mono" style="font-weight:800;color:var(--accent)">{{ a.code }}</td>
              <td><span class="badge"><span class="dot" :style="{ background: store.unitOfUid(a.unit_id)?.color }"></span>{{ a.unit?.name }}</span></td>
              <td>{{ { match: '🏀 比分', track: '🏃 成绩', eligibility: '🪪 资格' }[a.target_type] }}</td>
              <td class="ph" style="max-width:200px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">{{ appealBrief(a) }}</td>
              <td><span class="tag" :class="{ pending: 'o', reviewing: 'b', upheld: 'g', rejected: 'r', withdrawn: 'gray' }[a.status]">{{ { pending: '待受理', reviewing: '复核中', upheld: '已改判', rejected: '已驳回', withdrawn: '已撤案' }[a.status] }}</span></td>
              <td class="ph" style="max-width:220px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">{{ a.review_note || a.reason }}</td>
              <td class="ph" style="max-width:220px;overflow:hidden;text-overflow:ellipsis;white-space:nowrap">{{ appealImpact(a) }}</td>
              <td>{{ a.reviewer || '—' }}</td>
            </tr>
          </tbody>
        </table>
        <div v-if="!store.appeals.length" class="empty">暂无申诉记录</div>
      </div>
    </div>

    <!-- 项目明细 -->
    <div class="card mt">
      <div class="caption">🗂️ 赛事项目明细与规则</div>
      <div class="pad">
        <table>
          <thead><tr><th>项目</th><th>类别</th><th>赛制</th><th>场地</th><th>已完成/总场次</th><th>冠军归属</th></tr></thead>
          <tbody>
            <tr v-for="s in store.sports" :key="s.id">
              <td><b>{{ s.name }}</b></td>
              <td><span class="tag b">{{ s.category }}</span></td>
              <td>{{ s.format === 'roundrobin' ? '单循环积分制' : s.format === 'group_knockout' ? '小组赛 + 淘汰赛' : s.format === 'knockout' ? '单败淘汰' : '计时成绩' }}</td>
              <td>{{ s.venue }}</td>
              <td class="mono">{{ store.overview?.sportDone?.find(x=>x.id===s.id)?.done || 0 }}/{{ store.overview?.sportDone?.find(x=>x.id===s.id)?.total || 0 }}</td>
              <td class="ph">{{ s.finished ? '已产生' : '待结算' }}</td>
            </tr>
          </tbody>
        </table>
      </div>
    </div>
  </div>
</template>

<style scoped>
.hbar-duo { position: relative; height: 9px; border-radius: 20px; background: var(--bg2); overflow: hidden; display: flex; }
.hbar-duo i { display: block; height: 100%; border-radius: 20px; }
.hbar-duo i.d { background: var(--accent2); }
.hbar-duo i.u { background: var(--accent3); margin-left: 2px; }
.role-mini { background: var(--bg2); border-radius: 10px; padding: 8px 10px; }
.role-mini .hbar { height: 7px; border-radius: 20px; background: #e8edf3; overflow: hidden; margin-top: 5px; }
.role-mini .hbar i { display: block; height: 100%; border-radius: 20px; }
</style>