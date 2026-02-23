<script setup lang="ts">
import { ref, computed, watch, onMounted, nextTick, onUnmounted } from 'vue'

// HPCC Bottleneck tracking: c = floor(log_1.05(v))
const STEPS = [
  { id: 'S₁', util: 30000, encoded: 211 }, // 1.05^211 ≈ 29646
  { id: 'S₂', util: 75000, encoded: 230 }, // 1.05^230 ≈ 74737 (bottleneck)
  { id: 'S₃', util: 10000, encoded: 188 }, // 1.05^188 ≈ 9668
  { id: 'S₄', util: 50000, encoded: 221 }, // 1.05^221 ≈ 48296
]

// Determine dynamically which switches will write (update the max)
const writes: boolean[] = []
let currentMax = 0
for (let i = 0; i < STEPS.length; i++) {
  if (STEPS[i].encoded > currentMax) {
    writes.push(true)
    currentMax = STEPS[i].encoded
  } else {
    writes.push(false)
  }
}

const step = ref(-1)   // -1=not started  0-3=at switch  4=at sink

// ── Measure switch pixel positions after mount ──────────────────
const netEl   = ref<HTMLElement | null>(null)
const pktEl   = ref<HTMLElement | null>(null)
const swEls   = ref<HTMLElement[]>([])
const sinkEl  = ref<HTMLElement | null>(null)
const swPx    = ref<number[]>([])   
const sinkPx  = ref(0)

function measure() {
  if (!netEl.value || !pktEl.value) return
  const cRect = netEl.value.getBoundingClientRect()
  const pW    = pktEl.value.getBoundingClientRect().width

  swPx.value = swEls.value.map(el => {
    const r = el.getBoundingClientRect()
    return (r.left - cRect.left) + r.width / 2 - pW / 2
  })

  if (sinkEl.value) {
    const r = sinkEl.value.getBoundingClientRect()
    sinkPx.value = (r.left - cRect.left) + r.width / 2 - pW / 2
  }
}

// ── Keyboard Navigation ──────────────────────────────────────────
function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'ArrowRight' || e.key === 'Enter') {
    if (step.value < STEPS.length) {
      advance()
      e.preventDefault()
      e.stopPropagation() // Prevent Slidev from changing slides
    }
  } else if (e.key === 'ArrowLeft') {
    if (step.value > -1) {
      back()
      e.preventDefault()
      e.stopPropagation()
    }
  }
}

onMounted(() => {
  nextTick(measure)
  window.addEventListener('keydown', handleKeydown, { capture: true })
})

onUnmounted(() => {
  window.removeEventListener('keydown', handleKeydown, { capture: true })
})

// ── Packet position ──────────────────────────────────────────────
const pktLeft = computed(() => {
  if (step.value < 0)                  return '-140px'
  if (step.value >= STEPS.length)      return `${sinkPx.value}px`
  return `${swPx.value[step.value] ?? 0}px`
})

// ── Digest (Max Encoded Value) ───────────────────────────────────
const digest = computed(() => {
  if (step.value < 0) return 0
  let d = 0
  for (let i = 0; i <= Math.min(step.value, STEPS.length - 1); i++) {
    if (STEPS[i].encoded > d) d = STEPS[i].encoded
  }
  return d
})

function prevDigest(currentStep: number) {
  let d = 0
  for (let i = 0; i < currentStep; i++) {
    if (STEPS[i].encoded > d) d = STEPS[i].encoded
  }
  return d
}

const decodedValue = computed(() => {
  return Math.round(Math.pow(1.05, digest.value)).toLocaleString()
})

// True bottleneck: switch with highest encoded value
const trueBottleneck = computed(() => {
  return STEPS.reduce((max, sw) => sw.encoded > max.encoded ? sw : max)
})

// Multiplicative error on delivery
const errorPct = computed(() => {
  const decoded = Math.round(Math.pow(1.05, digest.value))
  return (Math.abs(trueBottleneck.value.util - decoded) / trueBottleneck.value.util * 100).toFixed(2)
})

// Per-switch: decode error for each encoded value
function swDecodeError(sw: typeof STEPS[0]) {
  const decoded = Math.round(Math.pow(1.05, sw.encoded))
  return (Math.abs(sw.util - decoded) / sw.util * 100).toFixed(2)
}

// Flash the digest field green ONLY on forward write
const digestFlash = ref(false)
watch(step, (newVal, oldVal) => {
  if (newVal > oldVal && newVal >= 0 && newVal < STEPS.length && writes[newVal]) {
    digestFlash.value = true
    setTimeout(() => (digestFlash.value = false), 700)
  }
})

// ── Controls ─────────────────────────────────────────────────────
function advance() { if (step.value < STEPS.length) step.value++ }
function back()    { if (step.value > -1) step.value-- }
function reset()   { step.value = -1 }

const btnLabel = computed(() => {
  if (step.value < 0)               return '▶  Launch'
  if (step.value < STEPS.length - 1) return `Next: ${STEPS[step.value + 1].id}  →`
  if (step.value === STEPS.length - 1) return 'Deliver  →'
  return '✓  Delivered'
})

// State helpers
const swState = (i: number) => {
  if (i > step.value)  return 'pending'
  if (i === step.value) return 'active'
  return writes[i] ? 'wrote' : 'skipped'
}
</script>

<template>
  <div class="trace">

    <div class="net" ref="netEl">

      <div
        ref="pktEl"
        class="packet"
        :class="{ visible: step >= 0 }"
        :style="{ left: pktLeft }"
      >
        <span class="plabel">INT_HDR</span>
        <span class="dig" :class="{ flash: digestFlash }">{{ digest }}</span>
      </div>

      <div class="src-node">SRC</div>
      <div class="wire"></div>

      <template v-for="(sw, i) in STEPS" :key="i">
        <div
          class="sw"
          :ref="(el) => { if (el) swEls[i] = el as HTMLElement }"
          :class="swState(i)"
        >
          <span class="sw-id">{{ sw.id }}</span>
          <span class="sw-util">{{ sw.util / 1000 }}k</span>
        </div>
        <div class="wire" :class="{ lit: step > i }"></div>
      </template>

      <div class="sink" ref="sinkEl" :class="{ lit: step >= STEPS.length }">SNK</div>
    </div>

    <div class="bubble-row">
      <div style="width:44px;flex-shrink:0"></div>
      <div style="flex:1"></div>
      <template v-for="(sw, i) in STEPS" :key="i">
        <div class="bslot">
          <Transition name="pop">
            <div v-if="step === i" class="bubble" :class="writes[i] ? 'bwrite' : 'bskip'">
              <div class="bline">v = <b>{{ sw.util.toLocaleString() }}</b></div>
              <div class="bline thresh">c = &lfloor;log<sub>1.05</sub>(v)&rfloor; = <b>{{ sw.encoded }}</b></div>
              <div class="bline thresh">decode: 1.05<sup>{{ sw.encoded }}</sup> &approx; {{ Math.round(Math.pow(1.05, sw.encoded)).toLocaleString() }} &nbsp;(err {{ swDecodeError(sw) }}%)</div>
              <div class="bdecision">{{ writes[i] ? '✓ NEW MAX' : '✗ LOWER (SKIP)' }}</div>
            </div>
          </Transition>
        </div>
        <div style="flex:1"></div>
      </template>
      <div style="width:44px;flex-shrink:0"></div>
    </div>

    <div class="status">
      <span v-if="step < 0"                class="s-idle">Press <b>Right Arrow</b> or Click ▶ to trace the bottleneck</span>
      <span v-else-if="step < STEPS.length" :class="writes[step] ? 's-write' : 's-skip'">
        <b>{{ STEPS[step].id }}</b> compares 8-bit encoded util:
        c = {{ STEPS[step].encoded }}
        {{ writes[step] ? '&gt;' : '&le;' }}
        {{ prevDigest(step) }} (current pkt max)
        &nbsp;→&nbsp;
        {{ writes[step]
            ? `Updates packet digest to ${STEPS[step].encoded}`
            : `Skips — digest remains ${digest}` }}
      </span>
      <span v-else class="s-done">
        Delivered &nbsp;·&nbsp; bottleneck digest = <b>{{ digest }}</b> (1 byte)
        &nbsp;·&nbsp; Decoded: 1.05<sup>{{ digest }}</sup> &approx; <b>{{ decodedValue }} Mbps</b>
        &nbsp;·&nbsp; True: <b>{{ trueBottleneck.util.toLocaleString() }} Mbps</b>
        &nbsp;·&nbsp; Error: <b>{{ errorPct }}%</b> &lt; 5% ✓
      </span>
    </div>

    <div class="ctrl">
      <button class="btn-back"  :disabled="step <= -1" @click="back">← Back</button>
      <button class="btn-reset" :disabled="step < 0" @click="reset">↺ Reset</button>
      <button class="btn-next"  :disabled="step >= STEPS.length" @click="advance">{{ btnLabel }}</button>
    </div>

  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; }

.trace {
  display: flex;
  flex-direction: column;
  gap: 2px;
  padding: 6px 0;
  font-family: 'JetBrains Mono', 'Fira Code', ui-monospace, monospace;
  font-size: 13px;
  width: 100%;
}

/* ── Network ── */
.net {
  position: relative;
  display: flex;
  align-items: center;
  height: 64px;
  overflow: visible;
}

/* Moving packet */
.packet {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  display: inline-flex;
  align-items: center;
  gap: 7px;
  background: #2d1b4e;
  border: 2px solid #a855f7;
  border-radius: 7px;
  padding: 5px 11px;
  z-index: 20;
  white-space: nowrap;
  opacity: 0;
  pointer-events: none;
  transition: left 0.55s cubic-bezier(0.4,0,0.2,1), opacity 0.25s;
}
.packet.visible { opacity: 1; }

.plabel { color: #c084fc; font-size: 11px; font-weight: 700; }

.dig {
  background: #1a0a2e;
  border: 1px solid #7c3aed;
  border-radius: 4px;
  padding: 1px 9px;
  color: #e9d5ff;
  font-weight: 700;
  min-width: 30px;
  text-align: center;
  transition: background 0.3s, border-color 0.3s, color 0.3s;
}
.dig.flash {
  background: #14532d;
  border-color: #4ade80;
  color: #86efac;
}

/* Nodes */
.src-node, .sink {
  width: 44px;
  height: 36px;
  border-radius: 18px;
  border: 2px solid #555;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 11px;
  font-weight: 700;
  color: #777;
  background: #1a1a2a;
  flex-shrink: 0;
  transition: border-color 0.4s, color 0.4s;
}
.sink.lit { border-color: #4ade80; color: #4ade80; }

.wire {
  flex: 1;
  height: 2px;
  background: #3a3a3a;
  transition: background 0.5s;
}
.wire.lit { background: #4ade80; }

/* Switch boxes */
.sw {
  width: 56px;
  height: 56px;
  border-radius: 9px;
  border: 2px solid #555;
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  gap: 2px;
  flex-shrink: 0;
  background: #1a1a2e;
  color: #666;
  transition: all 0.3s;
}
.sw-id { font-size: 13px; font-weight: 700; line-height: 1; }
.sw-util { font-size: 10px; font-weight: 400; opacity: 0.7; line-height: 1; }

.sw.active  {
  border-color: #60a5fa;
  background: #1e3a5f;
  color: #93c5fd;
  box-shadow: 0 0 16px rgba(96,165,250,0.45);
}
.sw.active .sw-util { opacity: 0.9; color: #bfdbfe; }

.sw.wrote   { border-color: #4ade80; background: #14532d; color: #86efac; }
.sw.skipped { border-color: #2d3748; color: #4a5568; }

/* ── Bubble row ── */
.bubble-row {
  display: flex;
  align-items: flex-start;
  height: 82px;
  overflow: visible;
}

.bslot {
  width: 56px;
  flex-shrink: 0;
  position: relative;
  display: flex;
  justify-content: center;
}

.bubble {
  position: absolute;
  top: 6px;
  left: 50%;
  transform: translateX(-50%);
  border: 2px solid;
  border-radius: 9px;
  padding: 6px 14px;
  font-size: 11px;
  white-space: nowrap;
  text-align: center;
  line-height: 1.65;
  z-index: 30;
}
.bwrite { background: #052e16; border-color: #4ade80; color: #bbf7d0; }
.bskip  { background: #2d1010; border-color: #f87171; color: #fecaca; }

.thresh     { opacity: 0.65; font-size: 10px; }
.bdecision  { font-weight: 700; font-size: 13px; margin-top: 2px; }
.bwrite .bdecision { color: #4ade80; }
.bskip  .bdecision { color: #f87171; }

/* ── Status ── */
.status {
  min-height: 28px;
  display: flex;
  align-items: center;
  justify-content: center;
  font-size: 13px;
  text-align: center;
  padding: 0 4px;
}
.s-idle  { color: #4b5563; }
.s-write { color: #86efac; }
.s-skip  { color: #94a3b8; }
.s-done  { color: #93c5fd; }

/* ── Controls ── */
.ctrl {
  display: flex;
  gap: 10px;
  justify-content: center;
  margin-top: 6px;
}

button {
  padding: 5px 18px;
  border-radius: 7px;
  border: 1px solid;
  cursor: pointer;
  font-family: inherit;
  font-size: 12px;
  transition: all 0.15s;
}
button:disabled { opacity: 0.3; cursor: not-allowed; }

.btn-back, .btn-reset {
  background: #1a1a2e;
  border-color: #374151;
  color: #6b7280;
}
.btn-back:hover:not(:disabled), .btn-reset:hover:not(:disabled) { 
  background: #252540; border-color: #555; color: #aaa; 
}

.btn-next {
  background: #2d1b4e;
  border-color: #7c3aed;
  color: #c4b5fd;
}
.btn-next:hover:not(:disabled) { background: #3d2b6e; border-color: #a855f7; }

/* Bubble pop transition */
.pop-enter-active { transition: all 0.2s ease-out; }
.pop-leave-active { transition: all 0.15s ease-in; }
.pop-enter-from, .pop-leave-to {
  opacity: 0;
  transform: translateX(-50%) translateY(-8px) scale(0.85);
}
</style>
