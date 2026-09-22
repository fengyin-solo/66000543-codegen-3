<template>
  <div class="panel">
    <div class="panel-head">
      <h4>📉 窗口日志量对比</h4>
      <div class="legend" title="高亮：偏离区间均值超过 1 个标准差（σ）的窗口">
        <span class="lg"><i class="dot normal"></i>正常</span>
        <span class="lg"><i class="dot high"></i>偏高</span>
        <span class="lg"><i class="dot low"></i>偏低</span>
        <span class="lg"><i class="line"></i>区间均值</span>
      </div>
    </div>
    <div class="chart-wrap">
      <div ref="chart" class="chart"></div>
      <div v-if="note" class="empty">{{ note }}</div>
    </div>
    <div v-if="!note" class="range-strip">
      <span class="range-info">
        W{{ view.start }}–W{{ view.end }} / 共{{ view.total }}窗 · 区间均值{{ view.mean }}条
        <em v-if="view.deviants" class="dev-tip">· {{ view.deviants }} 窗显著偏离</em>
      </span>
      <div class="chip-scroll">
        <span
          v-for="(c, i) in view.chips" :key="view.start + i"
          class="chip" :class="c.kind"
          :title="`窗口W${view.start + i}（日志 #${store.result!.windows[view.start + i].start}-${store.result!.windows[view.start + i].end - 1}）：${c.value} 条，偏离均值 ${c.delta > 0 ? '+' : ''}${c.delta}`"
        >W{{ view.start + i }}:{{ c.value }}</span>
      </div>
      <button v-if="view.zoomed" class="reset-btn" @click="resetZoom">重置区间</button>
    </div>
  </div>
</template>
<script setup lang="ts">
import { ref, reactive, watch, onMounted, onUnmounted } from 'vue'
import * as echarts from 'echarts'
import { useLogStore } from '../store/log'
import type { TimeWindow } from '../types'

const store = useLogStore(); const chart = ref<HTMLDivElement>(); let inst: echarts.ECharts|null=null

// 与其他图表一致：颜色、grid、生命周期均沿用统一写法
const COLOR = { normal:'#38bdf8', high:'#ef4444', low:'#f59e0b', avg:'#f97316' }
const note = ref('')
const applying = ref(false)
const view = reactive({
  start:0, end:0, total:0, mean:0, deviants:0, zoomed:false,
  chips: [] as { value:number; delta:number; kind:'normal'|'high'|'low' }[]
})
let viewStats = { mean:0, std:0 }

function kindOf(count:number) {
  if (viewStats.std > 1e-9 && count > viewStats.mean + viewStats.std) return 'high' as const
  if (viewStats.std > 1e-9 && count < viewStats.mean - viewStats.std) return 'low' as const
  return 'normal' as const
}

// 按当前窗口区间重算均值线、高亮与坐标轴范围
function applyView(ws:TimeWindow[], start:number, end:number) {
  view.start = start; view.end = end; view.total = ws.length
  view.zoomed = start !== 0 || end !== ws.length - 1
  const slice = ws.slice(start, end + 1).map(w => w.count)
  const mean = slice.length ? slice.reduce((a,b) => a+b, 0) / slice.length : 0
  const variance = slice.length ? slice.reduce((a,b) => a+(b-mean)**2, 0) / slice.length : 0
  const std = Math.sqrt(variance)
  viewStats = { mean, std }
  view.mean = Math.round(mean * 10) / 10

  const counts = ws.map(w => w.count)
  const max = Math.max(...slice, 1)
  view.chips = ws.slice(start, end + 1).map((w,i) => ({
    value: w.count, kind: kindOf(w.count),
    delta: Math.round((w.count - mean) * 10) / 10
  }))
  view.deviants = view.chips.filter(c => c.kind !== 'normal').length

  inst?.setOption({
    yAxis:{ min:0, max: Math.ceil(max * 1.15) },
    series:[{
      data: counts.map(v => ({ value:v, itemStyle:{ color: COLOR[kindOf(v)] } })),
      markLine:{ data:[{ yAxis: mean, name:'区间均值' }] }
    }]
  })
}

function onZoom(params: any) {
  if (applying.value || !store.result) return
  // 滚轮/拖拽 zoom 后，按区间重新统计并更新均值线与高亮
  const zoom = (inst?.getOption() as any).dataZoom?.[0]
  if (!zoom || params.batch) return
  const n = store.result.windows.length
  const start = Math.min(n - 1, Math.max(0, Math.floor(zoom.start / 100 * n)))
  const end = Math.min(n - 1, Math.max(start, Math.ceil(zoom.end / 100 * n) - 1))
  applying.value = true
  applyView(store.result.windows, start, end)
  applying.value = false
}

function resetZoom() {
  if (!store.result) return
  inst?.dispatchAction({ type:'dataZoom', start:0, end:100 })
}

function update() {
  if (!inst || !store.result) { note.value = '暂无窗口数据：请先生成日志'; inst?.clear(); return }
  const ws = store.result.windows
  // 窗口为空，或只有一两个窗口时无法构成有意义的均值对比，给出说明而不是画一根恒定的柱子
  if (ws.length === 0) { note.value = '暂无窗口数据：当前分析结果中没有时间窗口'; inst.clear(); return }
  if (ws.length === 1) { note.value = `仅 1 个窗口（W0，${ws[0].count} 条日志），无法与平均值对比，至少需要 3 个窗口`; inst.clear(); return }
  if (ws.length === 2) { note.value = `仅 2 个窗口（W0=${ws[0].count}，W1=${ws[1].count}），样本过少无法判断偏离，至少需要 3 个窗口`; inst.clear(); return }
  note.value = ''

  const counts = ws.map(w => w.count)
  inst.setOption({
    backgroundColor:'transparent',grid:{left:40,right:15,top:10,bottom:48},
    tooltip:{trigger:'axis',axisPointer:{type:'shadow'},textStyle:{fontSize:11},
      formatter:(ps:any) => {
        const p = Array.isArray(ps) ? ps[0] : ps
        const i = p.dataIndex, c = counts[i]
        const d = Math.round((c - viewStats.mean) * 10) / 10
        const k = kindOf(c)
        const tag = k === 'high' ? ' · <b style="color:#ef4444">显著偏高</b>'
          : k === 'low' ? ' · <b style="color:#f59e0b">显著偏低</b>' : ''
        return `窗口 W${i}（#${ws[i].start}–${ws[i].end - 1}）<br/>日志条数：<b>${c}</b>${tag}<br/>区间均值：${view.mean}（Δ ${d > 0 ? '+' : ''}${d}）`
      }},
    xAxis:{type:'category',data:ws.map((_,i)=>'W'+i),axisLabel:{color:'#94a3b8',fontSize:9}},
    yAxis:{type:'value',min:0,axisLabel:{color:'#94a3b8'}},
    dataZoom:[
      {type:'inside',xAxisIndex:0,filterMode:'filter'},
      {type:'slider',xAxisIndex:0,height:18,bottom:8,filterMode:'filter',
        backgroundColor:'#0f172a',fillerColor:'rgba(56,189,248,0.15)',borderColor:'#334155',
        dataBackground:{lineStyle:{color:'#475569'},areaStyle:{color:'#334155'}},
        selectedDataBackground:{lineStyle:{color:COLOR.avg},areaStyle:{color:'rgba(249,115,22,0.2)'}},
        handleStyle:{color:'#38bdf8'},moveHandleStyle:{color:'#64748b'},
        textStyle:{color:'#94a3b8',fontSize:9}}
    ],
    series:[{
      type:'bar',data:counts,barMaxWidth:24,itemStyle:{color:COLOR.normal},
      label:{show:true,position:'top',fontSize:9,color:'#cbd5e1',hideOverlap:true},
      markLine:{symbol:'none',silent:true,
        lineStyle:{color:COLOR.avg,type:'dashed',width:1.5},
        label:{color:COLOR.avg,fontSize:10,formatter:(p:any)=>'均值 '+Math.round(p.value*10)/10}}
    }],animation:false
  })

  applying.value = true
  applyView(ws, 0, ws.length - 1)
  applying.value = false
}

onMounted(()=>{
  if (chart.value) {
    inst = echarts.init(chart.value)
    inst.on('dataZoom', onZoom)
    update()
  }
})
watch(()=>store.result, update)
onUnmounted(()=>inst?.dispose())
</script>
<style scoped>
.panel{background:#1e293b;border-radius:8px;padding:12px;border:1px solid #334155}
.panel-head{display:flex;justify-content:space-between;align-items:center;gap:8px;margin-bottom:4px}
.panel h4{color:#38bdf8;font-size:13px}
.legend{display:flex;gap:8px;font-size:10px;color:#94a3b8}
.lg{display:inline-flex;align-items:center;gap:3px}
.dot{width:8px;height:8px;border-radius:2px;display:inline-block}
.dot.normal{background:#38bdf8}.dot.high{background:#ef4444}.dot.low{background:#f59e0b}
.line{width:12px;border-top:2px dashed #f97316;display:inline-block}
.chart-wrap{position:relative}
.chart{width:100%;height:196px}
.empty{position:absolute;inset:0;display:flex;align-items:center;justify-content:center;color:#64748b;font-size:12px;text-align:center;padding:0 24px}
.range-strip{display:flex;align-items:center;gap:8px;margin-top:2px}
.range-info{flex:none;font-size:10px;color:#94a3b8}
.dev-tip{color:#fca5a5;font-style:normal}
.chip-scroll{display:flex;gap:4px;overflow-x:auto;padding-bottom:2px}
.chip{flex:none;font-size:9px;padding:1px 5px;border-radius:3px;background:#0f172a;color:#7dd3fc;border:1px solid #334155;white-space:nowrap}
.chip.high{color:#fca5a5;border-color:#ef4444;background:#7f1d1d44}
.chip.low{color:#fcd34d;border-color:#f59e0b;background:#78350f44}
.reset-btn{flex:none;background:transparent;border:1px solid #475569;color:#94a3b8;font-size:10px;border-radius:3px;padding:1px 6px;cursor:pointer}
.reset-btn:hover{color:#e2e8f0;border-color:#94a3b8}
</style>
