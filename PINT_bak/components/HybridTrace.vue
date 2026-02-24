<script setup lang="ts">
import { ref, computed, onMounted, onUnmounted } from 'vue'

const PACKETS = [
  { name: 'p₁', mode: 'Baseline', switches: [0, 1, 0, 0, 0], digest: 'S₂', known: '{ S₂ }', note: 'Easy hop found quickly' },
  { name: 'p₂', mode: 'Baseline', switches: [0, 0, 0, 0, 1], digest: 'S₅', known: '{ S₂, S₅ }', note: 'Easy hop found quickly' },
  { name: 'p₃', mode: 'Baseline', switches: [1, 0, 0, 0, 0], digest: 'S₁', known: '{ S₁, S₂, S₅ }', note: 'Easy hop found quickly' },
  
  { name: 'p₄', mode: 'XOR', switches: [0, 0, 1, 1, 0], digest: 'S₃ ⊕ S₄', known: '{ S₁, S₂, S₅ }', 
    eqs: [{ text: 'p₄: S₃ ⊕ S₄', status: '(2 unknowns)' }],
    note: 'XOR layer adds equation, wait...' },
    
  { name: 'p₅', mode: 'Baseline', switches: [0, 1, 0, 0, 0], digest: 'S₂', known: '{ S₁, S₂, S₅ }', 
    eqs: [{ text: 'p₄: S₃ ⊕ S₄', status: '(2 unknowns)' }],
    note: 'Duplicate Baseline (wasted)' },
    
  { name: 'p₆', mode: 'XOR', switches: [0, 0, 0, 1, 0], digest: 'S₄', known: '{ S₁, S₂, S₄, S₅ }', 
    eqs: [
      { text: 'p₄: S₃ ⊕ S₄', status: '(2 unknowns)' },
      { text: 'p₆: S₄', status: '→ decode S₄ ✓', done: true }
    ],
    note: '1 unknown → decode S₄!' },
    
  { name: 'p₆ (back-sub)', mode: 'XOR', switches: [0, 0, 1, 0, 0], digest: 'sub S₄', known: '{ S₁, S₂, S₃, S₄, S₅ }', 
    eqs: [
      { text: 'p₄: S₃ ⊕ <s>S₄</s>', status: '→ decode S₃ ✓', done: true },
      { text: 'p₆: S₄', status: '✓', done: true }
    ],
    note: '✓ All 5 switches decoded in 6 packets!' }
]

const step = ref(-1)

function handleKeydown(e: KeyboardEvent) {
  if (e.key === 'ArrowRight' || e.key === 'Enter') {
    if (step.value < PACKETS.length - 1) { advance(); e.preventDefault(); e.stopPropagation(); }
  } else if (e.key === 'ArrowLeft') {
    if (step.value > -1) { back(); e.preventDefault(); e.stopPropagation(); }
  }
}

onMounted(() => window.addEventListener('keydown', handleKeydown))
onUnmounted(() => window.removeEventListener('keydown', handleKeydown))

function advance() { if (step.value < PACKETS.length - 1) step.value++ }
function back()    { if (step.value > -1) step.value-- }
function reset()   { step.value = -1 }

const current = computed(() => step.value >= 0 ? PACKETS[step.value] : null)
</script>

<template>
  <div class="trace">
    <div class="net">
      <div class="src-node">SRC</div>
      <div class="wire"></div>
      <template v-for="i in 5" :key="i">
        <div class="sw" :class="[
          { active: current?.switches[i-1] },
          current?.mode === 'XOR' && current?.switches[i-1] ? 'xor-mode' : ''
        ]">S{{ i }}</div>
        <div class="wire" :class="[
          { lit: current?.switches[i-1] },
          current?.mode === 'XOR' && current?.switches[i-1] ? 'xor-wire' : ''
        ]"></div>
      </template>
      <div class="sink lit">RCV</div>
    </div>

    <div class="dashboard hybrid-dash">
      
      <Transition name="fade" mode="out-in">
        <div v-if="current" :key="step" class="dash-content">
          <div class="pkt-card" :class="{ 'xor-card': current.mode === 'XOR' }">
            <div class="p-header">
              <span class="p-name">Packet {{ current.name }}</span>
              <span class="p-badge" :class="current.mode.toLowerCase()">{{ current.mode }} Layer</span>
            </div>
            <div class="p-dig" v-html="current.digest"></div>
            <div class="p-note" :class="{ success: current.note.includes('✓') }">{{ current.note }}</div>
            <div class="s-known mt-4 text-sm">{{ current.known }}</div>
          </div>
          <div class="state-card eq-card" v-if="current.eqs">
            <div class="s-title">Equations Map (XOR Layer)</div>
            <div class="eq-list">
              <div v-for="(eq, idx) in current.eqs" :key="idx" class="eq-line" :class="{ done: eq.done }">
                <span class="eq-left" v-html="eq.text"></span>
                <span class="eq-right">{{ eq.status }}</span>
              </div>
            </div>
          </div>
        </div>
      </Transition>
    </div>

    <div class="ctrl">
      <button class="btn-back" :disabled="step <= -1" @click="back">← Back</button>
      <button class="btn-reset" :disabled="step < 0" @click="reset">↺ Reset</button>
      <button class="btn-next" :disabled="step >= PACKETS.length - 1" @click="advance">Next Step →</button>
    </div>
  </div>
</template>

<style scoped>
*, *::before, *::after { box-sizing: border-box; }
.trace { display: flex; flex-direction: column; gap: 16px; font-family: 'JetBrains Mono', monospace; font-size: 13px; width: 100%; padding: 10px 0; }
.net { display: flex; align-items: center; height: 50px; }
.src-node, .sink { width: 44px; height: 36px; border-radius: 18px; border: 2px solid #555; display: flex; align-items: center; justify-content: center; font-size: 11px; font-weight: 700; color: #777; background: #1a1a2a; flex-shrink: 0; }
.sink.lit { border-color: #fcd34d; color: #fcd34d; background: #451a03; }
.wire { flex: 1; height: 2px; background: #3a3a3a; transition: background 0.3s; }

/* BASELINE colors (Purple/Pink) */
.wire.lit { background: #c084fc; }
.sw { width: 44px; height: 44px; border-radius: 9px; border: 2px solid #555; display: flex; align-items: center; justify-content: center; font-size: 13px; font-weight: 700; background: #1a1a2e; color: #666; transition: all 0.3s; flex-shrink: 0; }
.sw.active { border-color: #c084fc; background: #2d1b4e; color: #e9d5ff; box-shadow: 0 0 12px rgba(192,132,252,0.4); transform: scale(1.1); }

/* XOR colors (Blue) */
.wire.xor-wire { background: #60a5fa; }
.sw.active.xor-mode { border-color: #60a5fa; background: #1e3a8a; color: #bfdbfe; box-shadow: 0 0 12px rgba(96,165,250,0.4); }

.dashboard { min-height: 140px; background: #111118; border: 1px solid #333; border-radius: 8px; padding: 16px; display: flex; align-items: stretch; gap: 16px; }
.placeholder { margin: auto; color: #666; font-style: italic; }
.dash-content { display: flex; width: 100%; gap: 16px; }

.pkt-card { flex: 1; background: #1a1a2a; border-radius: 6px; padding: 12px; border-left: 3px solid #c084fc; transition: border-color 0.3s; }
.pkt-card.xor-card { border-left-color: #60a5fa; }
.p-header { display: flex; justify-content: space-between; align-items: center; margin-bottom: 6px; }
.p-name { color: #a1a1aa; font-size: 12px; }
.p-badge { font-size: 10px; padding: 2px 6px; border-radius: 4px; font-weight: bold; }
.p-badge.baseline { background: #3b0764; color: #d8b4fe; border: 1px solid #7e22ce; }
.p-badge.xor { background: #1e3a8a; color: #bfdbfe; border: 1px solid #2563eb; }

.p-dig { color: #fff; font-size: 16px; margin-bottom: 8px; font-weight: bold; }
.p-note { font-size: 12px; color: #94a3b8; }
.p-note.success { color: #4ade80; font-weight: bold; }
.s-known { font-size: 14px; color: #fde68a; font-weight: bold; margin-top: 12px; }

.eq-card { flex: 1; background: #1a1a2a; border-radius: 6px; padding: 12px; border-left: 3px solid #60a5fa; }
.s-title { color: #a1a1aa; font-size: 12px; margin-bottom: 8px; border-bottom: 1px solid #333; padding-bottom: 4px; }
.eq-line { display: flex; justify-content: space-between; padding: 4px 0; color: #94a3b8; }
.eq-line.done .eq-right { color: #4ade80; font-weight: bold; }
.eq-left :deep(s) { color: #ef4444; text-decoration-color: #ef4444; opacity: 0.8; }

.ctrl { display: flex; gap: 10px; justify-content: center; margin-top: 4px; }
button { padding: 5px 18px; border-radius: 7px; border: 1px solid; cursor: pointer; font-family: inherit; font-size: 12px; transition: all 0.15s; }
button:disabled { opacity: 0.3; cursor: not-allowed; }
.btn-back, .btn-reset { background: #1a1a2e; border-color: #374151; color: #6b7280; }
.btn-back:hover:not(:disabled), .btn-reset:hover:not(:disabled) { background: #252540; border-color: #555; color: #aaa; }
.btn-next { background: #451a03; border-color: #d97706; color: #fde68a; }
.btn-next:hover:not(:disabled) { background: #78350f; border-color: #f59e0b; }

.fade-enter-active, .fade-leave-active { transition: opacity 0.2s ease; }
.fade-enter-from, .fade-leave-to { opacity: 0; }
</style>
