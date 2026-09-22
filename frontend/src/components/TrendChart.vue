<template>
  <div class="panel">
    <h4>📉 窗口日志量趋势对比</h4>
    <div class="chart-wrap">
      <div ref="chart" class="chart"></div>
      <div v-if="hint" class="empty">{{ hint }}</div>
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref, watch, onMounted, onUnmounted } from 'vue'
import * as echarts from 'echarts'
import { useLogStore } from '../store/log'
import type { TimeWindow } from '../types'
const store = useLogStore(); const chart = ref<HTMLDivElement>(); let inst: echarts.ECharts|null=null
const hint = ref('')

// 按当前 dataZoom 百分比还原可见窗口下标区间（与 category 轴的换算方式一致）
function viewRange(start: number, end: number, n: number): [number, number] {
  const lo = Math.min(n - 1, Math.floor(start / 100 * n))
  const hi = Math.min(n, Math.max(lo + 1, Math.ceil(end / 100 * n)))
  return [lo, hi]
}

function buildOption(ws: TimeWindow[], lo = 0, hi = ws.length, resetZoom = false) {
  const counts = ws.map(w => w.count)
  const visible = counts.slice(lo, hi)
  const avg = visible.reduce((s, c) => s + c, 0) / visible.length
  const sd = Math.sqrt(visible.reduce((s, c) => s + (c - avg) ** 2, 0) / visible.length)
  // 与后端 3-sigma 判定同口径：偏离均值超过 1 个标准差的窗口高亮
  const deviated = (c: number) => sd > 1e-5 && Math.abs(c - avg) > sd
  const normal = counts.map((c, i) => ({
    value: c, itemStyle: { color: i >= lo && i < hi && deviated(c) ? 'rgba(0,0,0,0)' : '#38bdf8' }
  }))
  const highlighted = counts.map((c, i) => ({
    value: i >= lo && i < hi && deviated(c) ? c : '-', itemStyle: { color: '#ef4444' }
  }))
  const labelCfg = {
    show: true, position: 'top' as const, fontSize: 9, color: '#94a3b8',
    formatter: (p: any) => (p.value === '-' ? '' : String(p.value)), hideOverlap: true
  }
  const yMax = Math.max(...visible, 1)
  return {
    backgroundColor: 'transparent',
    grid: { left: 40, right: 15, top: 26, bottom: 46 },
    legend: { right: 0, top: 0, itemWidth: 10, itemHeight: 8, textStyle: { color: '#94a3b8', fontSize: 10 }, data: ['日志条数', '偏离平均'] },
    tooltip: { trigger: 'axis', axisPointer: { type: 'shadow' } as const,
      formatter: (ps: any) => {
        const i = ps[0].dataIndex; const w = ws[i]; const dev = deviated(counts[i])
        return `窗口 W${i}（日志 #${w.start}-#${w.end - 1}）<br/>日志条数：<b>${w.count}</b><br/>区间均值：${avg.toFixed(1)}${dev ? '<br/><span style="color:#ef4444">偏离平均</span>' : ''}`
      } },
    xAxis: { type: 'category', data: ws.map((_, i) => 'W' + i), axisLabel: { color: '#94a3b8', fontSize: 9 } },
    yAxis: { type: 'value', min: 0, max: Math.ceil(yMax * 1.15), axisLabel: { color: '#94a3b8' } },
    // filterMode:none 保留全部柱子，缩放只改变可视区间；区间切换时坐标轴/均值/高亮随下方 setOption 一起更新
    dataZoom: [{ type: 'slider', start: resetZoom ? 0 : lo / ws.length * 100, end: resetZoom ? 100 : hi / ws.length * 100,
      height: 14, bottom: 6, filterMode: 'none', borderColor: '#334155', backgroundColor: 'rgba(30,41,59,0.6)',
      fillerColor: 'rgba(56,189,248,0.12)', handleStyle: { color: '#38bdf8' }, textStyle: { color: '#64748b', fontSize: 9 } },
      { type: 'inside', filterMode: 'none' }],
    series: [
      { name: '日志条数', type: 'bar', data: normal, barCategoryGap: '30%', label: labelCfg,
        markLine: { symbol: 'none', silent: true,
          lineStyle: { color: '#f97316', type: 'dashed', width: 1.5 },
          label: { color: '#f97316', fontSize: 10, formatter: `均值 ${avg.toFixed(1)}` },
          data: [{ yAxis: avg, name: '平均值' }] } },
      { name: '偏离平均', type: 'bar', data: highlighted, barCategoryGap: '30%', barGap: '-100%',
        label: { ...labelCfg, color: '#fca5a5' }, tooltip: { show: false } }
    ], animation: false
  }
}

function update() {
  if (!inst) return
  const ws = store.result?.windows ?? []
  if (ws.length < 3) {
    inst.clear(); hint.value = !ws.length
      ? '暂无窗口数据，请先生成日志'
      : ws.length === 1
        ? '当前只有 1 个窗口（日志 1 条），无法进行均值对比，至少需要 3 个窗口'
        : `当前只有 ${ws.length} 个窗口（日志 ${ws.reduce((s, w) => s + w.count, 0)} 条），样本太少无法做偏离对比，至少需要 3 个窗口`
    return
  }
  hint.value = ''
  inst.setOption(buildOption(ws, 0, ws.length, true), true)
}

onMounted(() => {
  if (chart.value) {
    inst = echarts.init(chart.value)
    inst.on('dataZoom', () => {
      const ws = store.result?.windows ?? []
      if (!inst || ws.length < 3) return
      const opt = inst.getOption() as any
      const dz = opt.dataZoom?.[0]
      if (dz?.start == null || dz?.end == null) return
      const [lo, hi] = viewRange(dz.start, dz.end, ws.length)
      inst.setOption(buildOption(ws, lo, hi, false))
    })
    update()
  }
})
watch(() => store.result, update)
onUnmounted(() => inst?.dispose())
</script>
<style scoped>
.panel{background:#1e293b;border-radius:8px;padding:12px;border:1px solid #334155}
.panel h4{color:#38bdf8;font-size:13px;margin-bottom:4px}
.chart-wrap{position:relative}
.chart{width:100%;height:220px}
.empty{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;text-align:center;color:#64748b;font-size:12px;padding:0 24px}
</style>
