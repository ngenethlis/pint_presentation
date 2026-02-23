<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

// Latency of each switch (ms), index = switch 0..4 = S1..S5
const LATENCY = [2, 8, 3, 1, 12]

// Scripted packets: `winner` is the 0-based switch index whose value survives
// in the digest (uniform over hops due to reservoir sampling)
const PACKETS = [
  { name: 'p₁', winner: 2, note: 'S₃ writes at hop 3; S₄ and S₅ do not overwrite' },
  { name: 'p₂', winner: 4, note: 'S₅ writes at hop 5; last hop — cannot be overwritten' },
  { name: 'p₃', winner: 0, note: 'S₁ writes at hop 1; S₂–S₅ do not overwrite' },
  { name: 'p₄', winner: 4, note: 'S₅ survives again — high-latency hop appearing frequently' },
  { name: 'p₅', winner: 1, note: 'S₂ writes at hop 2; S₃–S₅ do not overwrite' },
  { name: 'p₆', winner: 3, note: 'S₄ writes at hop 4; S₅ does not overwrite this time' },
  { name: 'p₇', winner: 4, note: 'S₅ survives for the third time — P99 stable at 12ms' },
  { name: 'p₈', winner: 0, note: 'S₁ survives — distribution still filling in' },
]

const step   = ref(-1)
const rootEl = ref<HTMLElement | null>(null)

function isSlideActive(): boolean {
  if (!rootEl.value) return false
  const rect = rootEl.value.getBoundingClientRect()
  return rect.width > 0 && rect.right > 0 && rect.left < window.innerWidth
}

function handleKeydown(e: KeyboardEvent) {
  if (!isSlideActive()) return
  if (e.key === 'ArrowRight' || e.key === 'Enter') {
    if (step.value < PACKETS.length - 1) { advance(); e.preventDefault(); e.stopImmediatePropagation(); }
  } else if (e.key === 'ArrowLeft') {
    if (step.value > -1) { back(); e.preventDefault(); e.stopImmediatePropagation(); }
  }
}

onMounted(() => window.addEventListener('keydown', handleKeydown, { capture: true }))
onUnmounted(() => window.removeEventListener('keydown', handleKeydown, { capture: true }))

function advance() { if (step.value < PACKETS.length - 1) step.value++ }
function back()    { if (step.value > -1) step.value-- }
function reset()   { step.value = -1 }

const current = computed(() => step.value >= 0 ? PACKETS[step.value] : null)

// Collected latency values so far
const samples = computed(() => {
  const s: number[] = []
  for (let i = 0; i <= step.value; i++) s.push(LATENCY[PACKETS[i].winner])
  return s
})

// How many times each switch (0-indexed) won
const counts = computed(() => {
  const c = Array(5).fill(0)
  for (let i = 0; i <= step.value; i++) c[PACKETS[i].winner]++
  return c
})

const maxCount = computed(() => Math.max(1, ...counts.value))

// P99 from sorted samples
const p99 = computed(() => {
  if (samples.value.length === 0) return null
  const sorted = [...samples.value].sort((a, b) => a - b)
  const idx = Math.ceil(0.99 * sorted.length) - 1
  return sorted[Math.max(0, idx)]
})

// Switch index with the p99 latency value
const culprit = computed(() => {
  if (p99.value === null) return -1
  return LATENCY.indexOf(p99.value)
})

const stepLabel = computed(() => {
  if (step.value < 0) return `0 / ${PACKETS.length}`
  return `${step.value + 1} / ${PACKETS.length}`
})
</script>

<template>
  <div class="trace" ref="rootEl">

    <!-- Sample chips bar -->
    <div class="sample-bar">
      <span class="sb-label">Samples:</span>
      <span class="sb-chips">
        <span
          v-for="(ms, idx) in samples"
          :key="idx"
          class="chip"
          :class="{ 'chip-high': ms >= 10, 'chip-p99': ms === p99 }"
        >{{ ms }}ms</span>
        <span v-if="samples.length === 0" class="sb-empty">none yet</span>
      </span>
      <span class="sb-p99" v-if="p99 !== null">P99: <b>{{ p99 }}ms</b></span>
    </div>

    <!-- Network row: switches with latency labels -->
    <div class="net">
      <div class="src-node">SRC</div>
      <div class="wire"></div>
      <template v-for="i in 5" :key="i">
        <div class="sw-col">
          <div class="sw" :class="{
            active:  current?.winner === i-1,
            culprit: culprit === i-1 && culprit >= 0
          }">S{{ i }}</div>
          <div class="sw-lat" :class="{ 'lat-high': LATENCY[i-1] >= 10 }">{{ LATENCY[i-1] }}ms</div>
        </div>
        <div class="wire" :class="{ lit: current?.winner === i-1 }"></div>
      </template>
      <div class="sink lit">RCV</div>
    </div>

    <!-- Fixed info panel -->
    <div class="panel">
      <div class="panel-title">Reservoir sampling: hop $i$ writes with Pr = 1/i — any hop equally likely to survive; end-host feeds samples to KLL sketch for P99</div>

      <Transition name="fade" mode="out-in">
        <!-- Initial state -->
        <div v-if="!current" :key="'init'" class="panel-init">
          <div class="init-body">
            Each packet carries the latency of one surviving switch (chosen uniformly at random).
            Over many packets the distribution of per-hop latencies emerges — the sketch extracts P99.
            <div class="init-example">S5 = 12ms is the high-latency culprit — watch it dominate the tail</div>
          </div>
        </div>

        <!-- Active step -->
        <div v-else :key="step" class="dash-content">

          <!-- Left: current packet -->
          <div class="dash-left">
            <div class="left-label">Packet</div>
            <div class="left-name">{{ current.name }}</div>
            <div class="left-winner">
              <span class="winner-sw">S{{ current.winner + 1 }}</span>
              <span class="winner-arrow">→</span>
              <span class="winner-ms" :class="{ 'ms-high': LATENCY[current.winner] >= 10 }">
                {{ LATENCY[current.winner] }}ms
              </span>
            </div>
            <div class="left-note">{{ current.note }}</div>
            <div class="left-p99" v-if="p99 !== null">
              P99 estimate: <span class="p99-val">{{ p99 }}ms</span>
              <span class="p99-hint"> (S{{ culprit + 1 }})</span>
            </div>
          </div>

          <!-- Right: frequency bar chart -->
          <div class="dash-right">
            <div class="chart-title">Sample frequency per switch</div>
            <div class="chart">
              <div v-for="i in 5" :key="i" class="bar-col">
                <div class="bar-count">{{ counts[i-1] > 0 ? counts[i-1] : '' }}</div>
                <div class="bar-track">
                  <div
                    class="bar"
                    :class="{ 'bar-culprit': culprit === i-1 }"
                    :style="{ height: (counts[i-1] / maxCount * 52) + 'px' }"
                  ></div>
                </div>
                <div class="bar-sw" :class="{ 'lbl-culprit': culprit === i-1 }">S{{ i }}</div>
                <div class="bar-ms" :class="{ 'lbl-culprit': culprit === i-1 }">{{ LATENCY[i-1] }}ms</div>
              </div>
            </div>
            <div class="p99-banner" v-if="p99 !== null">
              P99 = {{ p99 }}ms &nbsp;→&nbsp; S{{ culprit + 1 }} is the bottleneck
            </div>
          </div>
        </div>
      </Transition>
    </div>

    <!-- Controls -->
    <div class="ctrl">
      <button class="btn-back"  :disabled="step <= -1" @click="back">← Back</button>
      <button class="btn-reset" :disabled="step < 0"   @click="reset" title="Reset">↺</button>
      <span class="step-counter">{{ stepLabel }}</span>
      <button class="btn-next"  :disabled="step >= PACKETS.length - 1" @click="advance">Next Packet →</button>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; }

.trace {
  display: flex;
  flex-direction: column;
  gap: 8px;
  padding: 6px 0;
  font-family: 'JetBrains Mono', monospace;
  font-size: 13px;
  width: 100%;
}

/* ── Sample chips bar ── */
.sample-bar {
  display: flex;
  align-items: center;
  gap: 8px;
  padding: 6px 10px;
  background: #0d0d18;
  border: 1px solid #1f2937;
  border-radius: 8px;
  flex-wrap: wrap;
  min-height: 38px;
}
.sb-label {
  font-size: 11px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  flex-shrink: 0;
}
.sb-chips { display: flex; gap: 4px; flex-wrap: wrap; flex: 1; }
.sb-empty { font-size: 11px; color: #374151; font-style: italic; }
.chip {
  font-size: 11px;
  font-weight: 700;
  padding: 2px 7px;
  border-radius: 4px;
  background: #1a1a2a;
  border: 1px solid #374151;
  color: #9ca3af;
}
.chip.chip-high   { background: #1c0a0a; border-color: #991b1b; color: #fca5a5; }
.chip.chip-p99    { border-color: #ef4444; }
.sb-p99 {
  font-size: 12px;
  color: #f87171;
  flex-shrink: 0;
  font-weight: 700;
}
.sb-p99 b { color: #ef4444; }

/* ── Network ── */
.net { display: flex; align-items: center; height: 62px; }
.src-node, .sink {
  width: 44px; height: 36px; border-radius: 18px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; font-weight: 700; color: #777; background: #1a1a2a; flex-shrink: 0;
}
.sink.lit { border-color: #fb923c; color: #fb923c; background: #1c0a00; }
.wire { flex: 1; height: 2px; background: #3a3a3a; transition: background 0.3s; align-self: center; margin-bottom: 14px; }
.wire.lit { background: #fb923c; }
.sw-col {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 2px;
  flex-shrink: 0;
}
.sw {
  width: 44px; height: 44px; border-radius: 9px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 13px; font-weight: 700; background: #1a1a2e; color: #666;
  transition: all 0.3s;
}
.sw.active  { border-color: #fb923c; background: #431407; color: #fed7aa; box-shadow: 0 0 12px rgba(251,146,60,0.45); transform: scale(1.1); }
.sw.culprit { border-color: #ef4444; }
.sw.active.culprit { border-color: #ef4444; background: #450a0a; color: #fca5a5; box-shadow: 0 0 14px rgba(239,68,68,0.5); }
.sw-lat { font-size: 10px; color: #4b5563; font-weight: 600; }
.sw-lat.lat-high { color: #f87171; }

/* ── Panel ── */
.panel {
  background: #111118;
  border: 1px solid #2d2d3a;
  border-radius: 10px;
  padding: 12px 16px;
  min-height: 110px;
  display: flex;
  flex-direction: column;
  gap: 8px;
}
.panel-title {
  font-size: 11px;
  color: #6b7280;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 6px;
  line-height: 1.4;
}
.panel-init {
  flex: 1;
  display: flex;
  align-items: center;
}
.init-body {
  color: #4b5563;
  font-size: 13px;
  font-style: italic;
  line-height: 1.6;
}
.init-example {
  margin-top: 8px;
  font-style: normal;
  color: #fca5a5;
  font-size: 13px;
  background: #1c0a0a;
  border: 1px solid #991b1b;
  border-radius: 6px;
  padding: 4px 12px;
  display: inline-block;
}

/* 2-column layout */
.dash-content { flex: 1; display: flex; gap: 12px; align-items: stretch; }

/* Left card */
.dash-left {
  flex: 1;
  background: #1c0a00;
  border: 1px solid #c2410c;
  border-radius: 8px;
  padding: 10px 14px;
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.left-label { font-size: 10px; color: #6b7280; text-transform: uppercase; letter-spacing: 0.06em; }
.left-name { font-size: 14px; font-weight: 700; color: #fdba74; }
.left-winner {
  display: flex;
  align-items: baseline;
  gap: 8px;
  margin: 4px 0;
}
.winner-sw { font-size: 18px; font-weight: 700; color: #fb923c; }
.winner-arrow { color: #4b5563; }
.winner-ms { font-size: 22px; font-weight: 700; color: #fed7aa; }
.winner-ms.ms-high { color: #fca5a5; }
.left-note { font-size: 12px; color: #94a3b8; line-height: 1.4; }
.left-p99 { margin-top: 4px; font-size: 12px; color: #6b7280; }
.p99-val { color: #ef4444; font-weight: 700; font-size: 14px; }
.p99-hint { color: #9ca3af; }

/* Right card: bar chart */
.dash-right {
  flex: 2;
  background: #0f0a1a;
  border: 1px solid #6d28d9;
  border-radius: 8px;
  padding: 10px 14px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.chart-title {
  font-size: 11px;
  color: #9ca3af;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 4px;
}
.chart {
  display: flex;
  gap: 10px;
  align-items: flex-end;
  flex: 1;
  padding-bottom: 2px;
}
.bar-col {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1px;
}
.bar-count {
  font-size: 11px;
  color: #9ca3af;
  font-weight: 700;
  height: 16px;
  line-height: 16px;
}
.bar-track {
  width: 100%;
  height: 52px;
  display: flex;
  align-items: flex-end;
}
.bar {
  width: 100%;
  background: #f59e0b;
  border-radius: 3px 3px 0 0;
  transition: height 0.4s ease;
  min-height: 0;
}
.bar.bar-culprit { background: #ef4444; }
.bar-sw { font-size: 11px; font-weight: 700; color: #6b7280; padding-top: 3px; }
.bar-ms { font-size: 10px; color: #4b5563; }
.lbl-culprit { color: #f87171 !important; }
.p99-banner {
  font-size: 12px;
  color: #f87171;
  font-weight: 700;
  text-align: center;
  padding: 3px 0;
  border-top: 1px solid #1f2937;
  margin-top: 2px;
}

/* ── Controls ── */
.ctrl {
  display: flex;
  align-items: center;
  gap: 8px;
  justify-content: center;
  margin-top: 4px;
}
.step-counter {
  font-size: 13px;
  color: #6b7280;
  min-width: 52px;
  text-align: center;
  font-variant-numeric: tabular-nums;
}
button {
  padding: 5px 14px;
  border-radius: 7px;
  border: 1px solid;
  cursor: pointer;
  font-family: inherit;
  font-size: 12px;
  transition: all 0.15s;
}
button:disabled { opacity: 0.3; cursor: not-allowed; }
.btn-back { background: #1a1a2e; border-color: #374151; color: #6b7280; }
.btn-back:hover:not(:disabled) { background: #252540; border-color: #555; color: #aaa; }
.btn-reset {
  background: transparent;
  border-color: transparent;
  color: #4b5563;
  padding: 5px 8px;
  font-size: 14px;
}
.btn-reset:hover:not(:disabled) { color: #9ca3af; }
.btn-next { background: #1c0a00; border-color: #c2410c; color: #fdba74; }
.btn-next:hover:not(:disabled) { background: #431407; border-color: #ea580c; }

.fade-enter-active { transition: opacity 0.2s ease; }
.fade-leave-active { transition: opacity 0.15s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
