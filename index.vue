<template>
  <div class="a">
    <header class="b">
      <h1>Cave Run</h1>
      <p>Score: <b>{{ s }}</b> | Best: <b>{{ h }}</b></p>
      <div class="hp-track">
        <div class="hp-bar" :style="`width:${hp}%`"></div>
      </div>
    </header>

    <main class="c">
      <!-- 3-Second Temporal Automatic Restart Delay Overlay Layout -->
      <div v-if="o" class="o">
        <h2 class="dead-title">SHIELDS CRUSHED</h2>
        <p>Re-cloning backup module soon...</p>
      </div>
      <canvas ref="g" width="320" height="360"></canvas>
    </main>

    <!-- Anchored Control Sheet for precise responsive delta tracking -->
    <footer class="f">
      <div class="pad-spacer"></div>
      <div class="ctrl-pad" ref="rPad" @touchstart.prevent="ts" @touchmove.prevent="tm" @touchend.prevent="te">
        <div class="btn-inner">SLIDE Y</div>
      </div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'

const s = ref(0), h = ref(0), o = ref(false), g = ref(null), hp = ref(100)
let cn, ctx, t, py = 180, cv = [], treats = [], au = null, spd = 2, sy = 0, count = 0

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
    osc.frequency.setValueAtTime(100, au.currentTime)
    gain.gain.setValueAtTime(0.15, au.currentTime)
    gain.gain.linearRampToValueAtTime(0.01, au.currentTime + 0.08)
    osc.start(); osc.stop(au.currentTime + 0.08)
  } else if (type === 'heal') {
    osc.type = 'sine'
    osc.frequency.setValueAtTime(400, au.currentTime)
    osc.frequency.exponentialRampToValueAtTime(800, au.currentTime + 0.1)
    gain.gain.setValueAtTime(0.1, au.currentTime)
    gain.gain.linearRampToValueAtTime(0.01, au.currentTime + 0.1)
    osc.start(); osc.stop(au.currentTime + 0.1)
  } else if (type === 'boom') {
    osc.type = 'triangle'
    osc.frequency.setValueAtTime(60, au.currentTime)
    osc.frequency.linearRampToValueAtTime(10, au.currentTime + 0.4)
    gain.gain.setValueAtTime(0.3, au.currentTime)
    gain.gain.linearRampToValueAtTime(0.01, au.currentTime + 0.4)
    osc.start(); osc.stop(au.currentTime + 0.4)
  }
}

const r = () => {
  s.value = 0; hp.value = 100; o.value = false; py = 180; spd = 2; cv = []; treats = []; count = 0
  for (let i = 0; i < 34; i++) {
    cv.push({ x: i * 10, top: 40, bot: 320 })
  }
}

const ts = (evt) => {
  initAudio()
  if (evt.touches.length) sy = evt.touches.clientY
}

// Fixed Delta Control System mapping touch height velocity changes directly to the Y position axis
const tm = (evt) => {
  if (o.value || !evt.touches.length) return
  const dy = evt.touches.clientY - sy
  
  // Clean proportional multiplier shifting player position vertically
  py = Math.max(12, Math.min(336, py + dy * 1.5))
  sy = evt.touches.clientY
}

const te = () => { sy = 0 }

const tick = () => {
  if (o.value) return
  ctx.fillStyle = '#111112'
  ctx.fillRect(0, 0, 320, 360)

  count++
  spd = 2 + (s.value * 0.03)

  // Cavern path vector generation equations
  cv.forEach(p => { p.x -= spd })
  treats.forEach(tr => { tr.x -= spd })

  if (cv.length && cv[0].x < -10) {
    cv.shift()
    s.value += 1
    const last = cv[cv.length - 1]
    
    // Wave calculations to swing, stretch, and skew terrain limits smoothly
    let midY = 180 + Math.sin(count * 0.03) * 50
    let thickness = 150 + Math.cos(count * 0.05) * 35 
    
    let nextTop = Math.max(10, midY - thickness / 2)
    let nextBot = Math.min(350, midY + thickness / 2)
    
    cv.push({ x: 320, top: nextTop, bot: nextBot })

    // Randomised item drop generator matrix
    if (Math.random() < 0.08 && treats.length < 3) {
      treats.push({ x: 320, y: nextTop + Math.random() * (nextBot - nextTop - 10) })
    }
  }

  // Draw procedural polygon borders segments 
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

  // Catching treat pickups calculation vectors
  treats.forEach((tr, idx) => {
    ctx.fillStyle = '#fff'
    ctx.beginPath()
    ctx.arc(tr.x, tr.y, 4, 0, Math.PI * 2)
    ctx.fill()

    if (tr.x < 0) treats.splice(idx, 1)

    // Bounding circle intersection metrics
    if (tr.x > 46 && tr.x < 66 && tr.y > py - 4 && tr.y < py + 16) {
      treats.splice(idx, 1)
      hp.value = Math.min(100, hp.value + 15)
      playSound('heal')
    }
  })

  // Cavern structure border hit tracking calculations
  const col = cv.find(p => p.x >= 48 && p.x <= 62)
  if (col && (py < col.top || (py + 12) > col.bot)) {
    if (count % 4 === 0) {
      hp.value -= 6
      playSound('hit')
    }
  }

  if (hp.value <= 0) {
    hp.value = 0; o.value = true; playSound('boom')
    if (s.value > h.value) h.value = s.value
    setTimeout(() => { r() }, 3000)
  }

  // Render player block asset
  ctx.fillStyle = '#fff'
  ctx.fillRect(50, py, 12, 12)
}

onMounted(() => {
  cn = g.value; ctx = cn.getContext('2d')
  r(); t = setInterval(tick, 1000 / 60)
  
  window.addEventListener('keydown', e => {
    initAudio()
    if (e.key === 'ArrowUp') py = Math.max(12, py - 16)
    if (e.key === 'ArrowDown') py = Math.min(336, py + 16)
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
.hp-track { width: 140px; height: 4px; background: #1c1c1e; margin: 0 auto; overflow: hidden; }
.hp-bar { height: 100%; background: #ef4444; width: 100%; transition: width 0.1s linear; }
.c { width: 100%; max-width: 320px; aspect-ratio: 320 / 360; position: relative; border: 2px solid #1c1c1e; box-shadow: 0 20px 40px rgba(0,0,0,0.6); background: #111112; }
canvas { display: block; width: 100%; height: 100%; }
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
