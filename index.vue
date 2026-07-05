<template>
  <div class="a">
    <header class="b">
      <h1>Cave Run</h1>
      <p>Score: <b>{{ s }}</b> | Best: <b>{{ h }}</b></p>
      <!-- Premium Integrated Health Shield Gauge -->
      <div class="hp-track">
        <div class="hp-bar" :style="`width:${hp}%`"></div>
      </div>
    </header>

    <main class="c">
      <!-- 3-Second Temporal Automatic Respawn Delay Overlay -->
      <div v-if="o" class="o">
        <h2 class="dead-title">CRITICAL ERROR</h2>
        <p>System destroyed. Re-cloning in progress...</p>
      </div>
      <canvas ref="g" width="320" height="360"></canvas>
    </main>

    <!-- Anchored Control Sheet for absolute layout tracking -->
    <footer class="f">
      <div class="pad-spacer"></div>
      <div class="ctrl-pad" ref="rPad" @touchstart.prevent="tm" @touchmove.prevent="tm">
        <div class="btn-inner">SLIDE Y</div>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const s = ref(0), h = ref(0), o = ref(false), g = ref(null), hp = ref(100)
let cn, ctx, t, py = 180, cv = [], au = null, spd = 2, dc = 0

const initAudio = () => {
  if (!au) au = new (window.AudioContext || window.webkitAudioContext)()
}

const playSound = (type) => {
  if (!au) return
  const osc = au.createOscillator()
  const gain = au.createGain()
  osc.connect(gain)
  gain.connect(au.destination)

  if (type === 'hit') {
    osc.type = 'sawtooth'
    osc.frequency.setValueAtTime(90, au.currentTime)
    gain.gain.setValueAtTime(0.2, au.currentTime)
    gain.gain.linearRampToValueAtTime(0.01, au.currentTime + 0.08)
    osc.start(); osc.stop(au.currentTime + 0.08)
  } else if (type === 'boom') {
    osc.type = 'triangle'
    osc.frequency.setValueAtTime(60, au.currentTime)
    osc.frequency.linearRampToValueAtTime(10, au.currentTime + 0.5)
    gain.gain.setValueAtTime(0.4, au.currentTime)
    gain.gain.linearRampToValueAtTime(0.01, au.currentTime + 0.5)
    osc.start(); osc.stop(au.currentTime + 0.5)
  }
}

const r = () => {
  s.value = 0; hp.value = 100; o.value = false; py = 180; spd = 2; cv = []
  for (let i = 0; i < 34; i++) {
    cv.push({ x: i * 10, top: 40, bot: 320 })
  }
}

// Controls the player block's exact Y position directly from the stick slider boundaries
const tm = (evt) => {
  if (o.value || !rPad.value || !evt.touches.length) return
  initAudio()
  const rect = rPad.value.getBoundingClientRect()
  const touchY = evt.touches[0].clientY
  
  // Normalise vertical input vectors relative to the controller circle's size scale
  const normY = (touchY - rect.top) / rect.height
  py = Math.max(10, Math.min(338, normY * 360))
}

const tick = () => {
  if (o.value) return
  ctx.fillStyle = '#111112'
  ctx.fillRect(0, 0, 320, 360)

  // Progressively accelerate speed scaling factors over temporal frames
  spd = 2 + (s.value * 0.04)

  cv.forEach(p => { p.x -= spd })
  if (cv[0] && cv[0].x < -10) {
    cv.shift()
    s.value += 1
    const last = cv[cv.length - 1]
    
    let change = Math.floor(Math.random() * 9) - 4
    let nextTop = Math.max(10, Math.min(90, last.top + change))
    let nextBot = Math.max(250, Math.min(350, last.bot + change))
    
    if (nextBot - nextTop < 120) {
      nextTop -= 8; nextBot += 8
    }
    
    cv.push({ x: 320, top: nextTop, bot: nextBot })
  }

  ctx.fillStyle = '#1c1c1e'
  ctx.beginPath()
  ctx.moveTo(cv[0]?.x || 0, 0)
  cv.forEach(p => ctx.lineTo(p.x, p.top))
  ctx.lineTo(320, 0)
  ctx.fill()

  ctx.beginPath()
  ctx.moveTo(cv[0]?.x || 0, 360)
  cv.forEach(p => ctx.lineTo(p.x, p.bot))
  ctx.lineTo(320, 360)
  ctx.fill()

  // Track overlapping damage intersections profile metrics
  const col = cv.find(p => p.x >= 45 && p.x <= 65)
  if (col && (py < col.top || (py + 12) > col.bot)) {
    dc++
    if (dc % 5 === 0) {
      hp.value -= 8
      playSound('hit')
    }
  }

  // Handle total destruction conditions
  if (hp.value <= 0) {
    hp.value = 0
    o.value = true
    playSound('boom')
    if (s.value > h.value) h.value = s.value
    
    // Automatically trigger systemic reconstruction cycles after 3 seconds
    setTimeout(() => { r() }, 3000)
  }

  ctx.fillStyle = '#fff'
  ctx.fillRect(50, py, 12, 12)
}

onMounted(() => {
  cn = g.value; ctx = cn.getContext('2d')
  r(); t = setInterval(tick, 1000 / 60)
  
  // Desktop keyboard precise scrolling fallback controls
  window.addEventListener('keydown', e => {
    initAudio()
    if (e.key === 'ArrowUp') py = Math.max(10, py - 12)
    if (e.key === 'ArrowDown') py = Math.min(338, py + 12)
  })
})
onUnmounted(() => clearInterval(t))
</script>

<style scoped>
.a { width: 100vw; height: 100vh; background: #000; color: #fff; display: flex; flex-direction: column; align-items: center; justify-content: space-between; padding: 16px; font-family: sans-serif; box-sizing: border-box; user-select: none; overflow: hidden; }
.b { text-align: center; margin-bottom: 5px; width: 100%; }
h1 { margin: 0; font-size: 20px; text-transform: uppercase; color: #e2e8f0; letter-spacing: 1.5px; }
p { margin: 4px 0 8px 0; color: #71717a; font-size: 13px; }
b { color: #fff; }

/* Structural Health Bar Elements styling overrides */
.hp-track { width: 140px; height: 4px; background: #1c1c1e; margin: 0 auto; overflow: hidden; border-radius: 0; }
.hp-bar { height: 100%; background: #ef4444; width: 100%; transition: width 0.1s linear; }

.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 360; position: relative; border: 2px solid #1c1c1e; box-shadow: 0 20px 40px rgba(0,0,0,0.6); background: #111112; }
canvas { display: block; width: 100%; height: 100%; }

/* Crash Overlay Sheet UI changes */
.o { position: absolute; top: 0; left: 0; width: 100%; height: 100%; background: rgba(0,0,0,0.98); display: flex; flex-direction: column; align-items: center; justify-content: center; z-index: 10; }
.dead-title { color: #ef4444; margin: 0; font-size: 18px; letter-spacing: 2px; text-transform: uppercase; animation: flash 0.5s steps(2, start) infinite; }
.o p { color: #71717a; font-size: 11px; margin-top: 8px; text-transform: uppercase; letter-spacing: 0.5px; }

@keyframes flash { to { visibility: hidden; } }

.f { width: 100%; max-width: 320px; display: flex; justify-content: space-between; padding: 16px 0; box-sizing: border-box; }
.pad-spacer { flex: 1; }
.ctrl-pad { width: 76px; height: 76px; background: #111112; border: 2px solid #2c2c2e; border-radius: 50%; display: flex; align-items: center; justify-content: center; touch-action: none; cursor: pointer; }
.ctrl-pad:active { background: #1c1c1e; border-color: #ef4444; }
.btn-inner { font-size: 9px; font-weight: 800; color: #71717a; letter-spacing: 0.5px; text-transform: uppercase; pointer-events: none; }
.ctrl-pad:active .btn-inner { color: #fff; }

@media (max-width: 360px) { .c { max-width: 90vw; } .f { max-width: 90vw; } }
</style>
