<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

const PACKETS = [
  { name: 'p₁', mode: 'Baseline', switches: [0, 1, 0, 0, 0], digest: 'S₂', known: '{ S2 }',
    note: 'Baseline layer: single switch ID recorded directly' },

  { name: 'p₂', mode: 'Baseline', switches: [0, 0, 0, 0, 1], digest: 'S₅', known: '{ S2, S5 }',
    note: 'Another isolated hop recovered' },

  { name: 'p₃', mode: 'Baseline', switches: [1, 0, 0, 0, 0], digest: 'S₁', known: '{ S1, S2, S5 }',
    note: '3 isolated hops recovered' },

  { name: 'p₄', mode: 'XOR', switches: [0, 0, 1, 1, 0], digest: 'S₃ ⊕ S₄', known: '{ S1, S2, S5 }',
    eqs: [{ text: 'p₄: S₃ ⊕ S₄', status: '2 unknowns — wait' }],
    note: 'XOR layer: adds equation to system, not yet solvable' },

  { name: 'p₅', mode: 'Baseline', switches: [0, 1, 0, 0, 0], digest: 'S₂', known: '{ S1, S2, S5 }',
    eqs: [{ text: 'p₄: S₃ ⊕ S₄', status: '2 unknowns — wait' }],
    note: 'Duplicate Baseline — wasted packet' },

  { name: 'p₆', mode: 'XOR', switches: [0, 0, 0, 1, 0], digest: 'S₄', known: '{ S1, S2, S4, S5 }',
    eqs: [
      { text: 'p₄: S₃ ⊕ S₄', status: '2 unknowns' },
      { text: 'p₆: S₄', status: '1 unknown → solve!', done: true }
    ],
    note: '1 unknown → decode S₄ directly' },

  { name: 'back-sub into p₄', mode: 'XOR', switches: [0, 0, 1, 0, 0], digest: 'sub S₄ → S₃', known: '{ S1, S2, S3, S4, S5 }',
    eqs: [
      { text: 'p₄: S₃ ⊕ <s>S₄</s>', status: 'substitute S₄ → solve S₃', done: true },
      { text: 'p₆: S₄',              status: '✓ solved', done: true }
    ],
    note: '✓ All 5 switches decoded in 6 packets!' }
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

// Which switch indices (0-based) are known
const knownSwitches = computed(() => {
  const known = new Set<number>()
  if (!current.value) return known
  const matches = current.value.known.matchAll(/S(\d)/g)
  for (const m of matches) known.add(parseInt(m[1]) - 1)
  return known
})

const knownCount = computed(() => knownSwitches.value.size)

const stepLabel = computed(() => {
  if (step.value < 0) return `0 / ${PACKETS.length}`
  return `${step.value + 1} / ${PACKETS.length}`
})
</script>

<template>
  <div class="trace" ref="rootEl">

    <!-- Progress bar: recovered switches -->
    <div class="progress-bar">
      <div class="pb-label">Switches recovered:</div>
      <div class="pb-boxes">
        <div
          v-for="i in 5"
          :key="i"
          class="pb-box"
          :class="{ known: knownSwitches.has(i-1) }"
        >S{{ i }}</div>
      </div>
      <div class="pb-count">{{ knownCount }} / 5</div>
    </div>

    <!-- Network row -->
    <div class="net">
      <div class="src-node">SRC</div>
      <div class="wire"></div>
      <template v-for="i in 5" :key="i">
        <div class="sw" :class="[
          current?.switches[i-1] ? 'active' : '',
          current?.mode === 'XOR' && current?.switches[i-1] ? 'xor-mode' : ''
        ]">S{{ i }}</div>
        <div class="wire" :class="[
          current?.switches[i-1] ? 'lit' : '',
          current?.mode === 'XOR' && current?.switches[i-1] ? 'xor-wire' : ''
        ]"></div>
      </template>
      <div class="sink lit">RCV</div>
    </div>

    <!-- Fixed info panel: 2-column dashboard -->
    <div class="panel">
      <div class="panel-title">Hybrid scheme: Baseline layer recovers isolated hops; XOR layer recovers multi-hop paths via 𝔽₂ back-substitution</div>

      <Transition name="fade" mode="out-in">
        <!-- Initial state -->
        <div v-if="!current" :key="'init'" class="panel-init">
          <div class="init-body">
            Hybrid combines two layers: <b>Baseline</b> (single switch ID per packet) and <b>XOR</b>
            (XOR digest of all traversed switches). Baseline finds isolated hops quickly;
            XOR solves multi-hop paths via Gaussian elimination over 𝔽₂.
            <div class="init-example">Goal: recover all 5 switches — Hybrid needs ~6 packets vs. Baseline's ~12</div>
          </div>
        </div>

        <!-- Active step: 2-col dashboard -->
        <div v-else :key="step" class="dash-content">

          <!-- Left: packet info -->
          <div class="dash-left">
            <div class="left-header">
              <div class="left-label">Packet</div>
              <span class="mode-badge" :class="current.mode.toLowerCase()">{{ current.mode }} layer</span>
            </div>
            <div class="left-name">{{ current.name }}</div>
            <div class="left-digest mono" v-html="current.digest"></div>
            <div class="left-note" :class="{
              'note-success': current.note.includes('✓'),
              'note-warn': current.note.includes('wasted') || current.note.includes('Duplicate')
            }">{{ current.note }}</div>
            <div class="left-known">Known: <span class="mono known-val">{{ current.known }}</span></div>
          </div>

          <!-- Right: XOR equation system -->
          <div class="dash-right">
            <div class="eq-title">XOR Equation System</div>
            <div v-if="current.eqs" class="eq-list">
              <div
                v-for="(eq, idx) in current.eqs"
                :key="idx"
                class="eq-line"
                :class="{ done: eq.done, pending: !eq.done }"
              >
                <span class="eq-text mono" v-html="eq.text"></span>
                <span class="eq-status">{{ eq.status }}</span>
              </div>
            </div>
            <div v-else class="eq-empty">
              No XOR equations yet — Baseline layer only
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
      <button class="btn-next"  :disabled="step >= PACKETS.length - 1" @click="advance">Next Step →</button>
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

/* ── Progress Bar ── */
.progress-bar {
  display: flex;
  align-items: center;
  gap: 10px;
  padding: 6px 10px;
  background: #0d0d18;
  border: 1px solid #1f2937;
  border-radius: 8px;
}
.pb-label {
  font-size: 11px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.05em;
  flex-shrink: 0;
}
.pb-boxes { display: flex; gap: 6px; flex: 1; }
.pb-box {
  width: 36px; height: 28px; border-radius: 6px;
  border: 2px solid #374151;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; font-weight: 700;
  color: #4b5563; background: #1a1a2a;
  transition: all 0.35s;
}
.pb-box.known { border-color: #4ade80; background: #052e16; color: #4ade80; }
.pb-count { font-size: 13px; font-weight: 700; color: #9ca3af; min-width: 36px; text-align: right; }

/* ── Network ── */
.net { display: flex; align-items: center; height: 50px; }
.src-node, .sink {
  width: 44px; height: 36px; border-radius: 18px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 11px; font-weight: 700; color: #777; background: #1a1a2a; flex-shrink: 0;
}
.sink.lit { border-color: #fcd34d; color: #fcd34d; background: #451a03; }
.wire { flex: 1; height: 2px; background: #3a3a3a; transition: background 0.3s; }
.wire.lit { background: #c084fc; }
.wire.xor-wire { background: #60a5fa; }
.sw {
  width: 44px; height: 44px; border-radius: 9px; border: 2px solid #555;
  display: flex; align-items: center; justify-content: center;
  font-size: 13px; font-weight: 700; background: #1a1a2e; color: #666;
  transition: all 0.3s; flex-shrink: 0;
}
/* Baseline = purple */
.sw.active { border-color: #c084fc; background: #2d1b4e; color: #e9d5ff; box-shadow: 0 0 12px rgba(192,132,252,0.4); transform: scale(1.1); }
/* XOR = blue (overrides active) */
.sw.active.xor-mode { border-color: #60a5fa; background: #1e3a8a; color: #bfdbfe; box-shadow: 0 0 12px rgba(96,165,250,0.4); }

/* ── Info Panel ── */
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
.init-body b { color: #9ca3af; font-style: normal; }
.init-example {
  margin-top: 8px;
  font-style: normal;
  color: #fde68a;
  font-size: 13px;
  background: #451a03;
  border: 1px solid #d97706;
  border-radius: 6px;
  padding: 4px 12px;
  display: inline-block;
}

/* 2-column dashboard */
.dash-content {
  flex: 1;
  display: flex;
  gap: 12px;
  align-items: stretch;
}

/* Left card */
.dash-left {
  flex: 1;
  border-radius: 8px;
  padding: 10px 14px;
  display: flex;
  flex-direction: column;
  gap: 4px;
  border: 1px solid;
  transition: background 0.3s, border-color 0.3s;
  background: #1a1228;
  border-color: #7c3aed;
}
.left-header {
  display: flex;
  align-items: center;
  justify-content: space-between;
  gap: 8px;
}
.left-label {
  font-size: 10px;
  color: #6b7280;
  text-transform: uppercase;
  letter-spacing: 0.06em;
}
.mode-badge {
  font-size: 10px;
  padding: 2px 7px;
  border-radius: 4px;
  font-weight: 700;
  flex-shrink: 0;
}
.mode-badge.baseline { background: #3b0764; color: #d8b4fe; border: 1px solid #7e22ce; }
.mode-badge.xor      { background: #1e3a8a; color: #bfdbfe; border: 1px solid #2563eb; }

.left-name {
  font-size: 14px;
  font-weight: 700;
  color: #e9d5ff;
}
.left-digest {
  font-size: 16px;
  font-weight: 700;
  color: #fff;
  margin: 2px 0;
}
.left-note {
  font-size: 12px;
  color: #94a3b8;
  line-height: 1.4;
}
.left-note.note-success { color: #4ade80; font-weight: 700; }
.left-note.note-warn    { color: #f87171; }
.left-known {
  margin-top: 4px;
  font-size: 12px;
  color: #6b7280;
}
.known-val { color: #fde68a; }

/* Right card: equations */
.dash-right {
  flex: 2;
  background: #130a2a;
  border: 1px solid #5b21b6;
  border-radius: 8px;
  padding: 10px 14px;
  display: flex;
  flex-direction: column;
  gap: 6px;
}
.eq-title {
  font-size: 11px;
  color: #9ca3af;
  text-transform: uppercase;
  letter-spacing: 0.06em;
  border-bottom: 1px solid #1f2937;
  padding-bottom: 4px;
}
.eq-list {
  display: flex;
  flex-direction: column;
  gap: 4px;
}
.eq-line {
  display: flex;
  align-items: baseline;
  justify-content: space-between;
  gap: 8px;
  padding: 4px 6px;
  border-radius: 5px;
}
.eq-line.pending { background: #1a1028; }
.eq-line.done    { background: #071a0e; }

.eq-text {
  font-size: 13px;
  color: #c4b5fd;
  flex-shrink: 0;
}
.eq-line.done .eq-text { color: #86efac; }
.eq-text :deep(s) { color: #ef4444; text-decoration-color: #ef4444; opacity: 0.8; }

.eq-status {
  font-size: 11px;
  color: #6b7280;
  text-align: right;
  flex: 1;
}
.eq-line.done .eq-status { color: #4ade80; font-weight: 700; }

.eq-empty {
  flex: 1;
  display: flex;
  align-items: center;
  font-size: 12px;
  color: #4b5563;
  font-style: italic;
}

.mono { font-family: 'JetBrains Mono', 'Fira Code', ui-monospace, monospace; }

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
.btn-next { background: #451a03; border-color: #d97706; color: #fde68a; }
.btn-next:hover:not(:disabled) { background: #78350f; border-color: #f59e0b; }

.fade-enter-active { transition: opacity 0.2s ease; }
.fade-leave-active { transition: opacity 0.15s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
