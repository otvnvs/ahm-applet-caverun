<template>
  <div class="a" @touchstart="ts" @touchmove="tm" @touchend="te">
    <header class="b">
      <h1>Cave</h1>
      <p>S:<b>{{ s }}</b>|H:<b>{{ h }}%</b></p>
      <div class="k"><div class="g" :style="`width:${h}%`"></div></div>
    </header>
    <main class="c">
      <div v-if="o" class="o"><h2>{{ h<=0?'DESTROYED':'REBOOTING' }}</h2></div>
      <canvas ref="v" width="320" height="340"></canvas>
    </main>
    <footer class="f">
      <div class="p" id="l">FIRE</div>
      <div class="p" id="r">STICK</div>
    </footer>
  </div>
</template>

<script setup>
import { ref, onMounted, onUnmounted } from 'vue'
const s=ref(0), h=ref(100), o=ref(false), v=ref(null)
let cx, ctx, t, px=50, py=170, cv=[], tr=[], en=[], b=[], au=null, vx=0, vy=0, n=0
const audio=(f,t,g,d)=>{
  if(!au)au=new(window.AudioContext||window.webkitAudioContext)()
  let o=au.createOscillator(),q=au.createGain()
  o.type=t;o.frequency.setValueAtTime(f,au.currentTime)
  q.gain.setValueAtTime(g,au.currentTime);q.gain.linearRampToValueAtTime(0,au.currentTime+d)
  o.connect(q);q.connect(au.destination);o.start();o.stop(au.currentTime+d)
}
const r=()=>{ s.value=0;h.value=100;o.value=false;px=50;py=170;vx=0;vy=0;cv=[];tr=[];en=[];b=[];n=0;for(let i=0;i<34;i++)cv.push({x:i*10,t:30,b:310}) }
const ts=(e)=>{
  if(!cx||o.value)return
  let r=cx.getBoundingClientRect()
  for(let t of e.touches){
    let x=t.clientX-r.left
    if(x<160&&b.length<4){b.push({x:px+12,y:py+6});audio(600,'sawtooth',0.1,0.08)}
  }
}
const tm=(e)=>{
  if(!cx||o.value)return
  let r=cx.getBoundingClientRect()
  for(let t of e.touches){
    let x=t.clientX-r.left, y=t.clientY-r.top
    if(x>=160){
      // Calibrated sensitivity matrix: reduced maximum deflection multiplier to 2.2 for steady navigation
      vx=Math.max(-1,Math.min(1,(x-240)/45))*2.2
      vy=Math.max(-1,Math.min(1,(y-(r.height-50))/45))*2.2
    }
  }
}
const te=(e)=>{if(!e.touches.length){vx=0;vy=0}}
const tick=()=>{
  if(o.value)return
  ctx.fillStyle='#111';ctx.fillRect(0,0,320,340)
  n++;let speed=5+(s.value*0.05)
  px=Math.max(10,Math.min(290,px+vx));py=Math.max(10,Math.min(315,py+vy))
  cv.forEach(p=>p.x-=speed);tr.forEach(t=>t.x-=speed);en.forEach(e=>e.x-=speed*1.2);b.forEach(b=>b.x+=8)
  b=b.filter(i=>i.x<320)
  if(cv.length&&cv[0].x<-10){
    cv.shift();s.value++
    let last=cv[cv.length-1], mY=170+Math.sin(n*0.04)*60, th=130+Math.cos(n*0.06)*30
    let nT=Math.max(10,mY-th/2), nB=Math.min(330,mY+th/2)
    cv.push({x:320,t:nT,b:nB})
    if(Math.random()<0.12&&tr.length<3)tr.push({x:320,y:nT+Math.random()*(nB-nT-10)})
    if(Math.random()<0.15&&en.length<4)en.push({x:320,y:nT+Math.random()*(nB-nT-15)})
  }
  ctx.fillStyle='#1c1c1e';ctx.beginPath();ctx.moveTo(cv[0]?.x||0,0);cv.forEach(p=>ctx.lineTo(p.x,p.t));ctx.lineTo(320,0);ctx.fill()
  ctx.beginPath();ctx.moveTo(cv[0]?.x||0,340);cv.forEach(p=>ctx.lineTo(p.x,p.b));ctx.lineTo(320,340);ctx.fill()
  b.forEach(i=>{ctx.fillStyle='#fff';ctx.fillRect(i.x,i.y,6,2)})
  tr.forEach((t,i)=>{
    ctx.fillStyle='#fff';ctx.beginPath();ctx.arc(t.x,t.y,4,0,Math.PI*2);ctx.fill()
    if(t.x>px&&t.x<px+12&&t.y>py&&t.y<py+12){tr.splice(i,1);h.value=Math.min(100,h.value+15);audio(440,'sine',0.1,0.1)}
  });tr=tr.filter(t=>t.x>0)
  en.forEach((e,i)=>{
    ctx.fillStyle='#ef4444';ctx.fillRect(e.x,e.y,14,10)
    b.forEach((bl,bi)=>{
      if(bl.x>e.x&&bl.x<e.x+14&&bl.y>e.y&&bl.y<e.y+10){en.splice(i,1);b.splice(bi,1);s.value+=10;audio(150,'triangle',0.2,0.15)}
    })
    if(e.x>px&&e.x<px+12&&e.y>py-10&&e.y<py+12){en.splice(i,1);h.value-=20;audio(100,'sawtooth',0.2,0.1)}
  });en=en.filter(e=>e.x>0)
  let c=cv.find(p=>p.x>=px&&p.x<=px+12)
  if(c&&(py<c.t||py+12>c.b)&&n%4===0){h.value-=5;audio(90,'sawtooth',0.15,0.05)}
  if(h.value<=0){h.value=0;o.value=true;audio(50,'triangle',0.4,0.5);setTimeout(r,3000)}
  ctx.fillStyle='#fff';ctx.fillRect(px,py,12,12)
}
onMounted(() => { cx=v.value;ctx=cx.getContext('2d');r();t=setInterval(tick,16) })
onUnmounted(() => clearInterval(t))
</script>

<style scoped>
.a{width:100vw;height:100vh;background:#000;color:#fff;display:flex;flex-direction:column;align-items:center;justify-content:space-between;padding:12px;box-sizing:border-box;font-family:sans-serif;user-select:none;overflow:hidden;}
.b{text-align:center;width:100%;}h1{margin:0;font-size:18px;text-transform:uppercase;letter-spacing:1px;}.k{width:120px;height:4px;background:#222;margin:4px auto;}.g{height:100%;background:#ef4444;transition:width .1s;}
p{margin:2px;color:#71717a;font-size:12px;}b{color:#fff;}
.c{width:100%;max-width:320px;aspect-ratio:320/340;position:relative;border:2px solid #1c1c1e;background:#111;}
canvas{display:block;width:100%;height:100%;}
.o{position:absolute;top:0;left:0;width:100%;height:100%;background:rgba(0,0,0,.95);display:flex;align-items:center;justify-content:center;}h2{color:#ef4444;font-size:16px;}
.f{width:100%;max-width:320px;display:flex;justify-content:space-between;padding:8px 0;}
.p{width:70px;height:70px;background:#111;border:2px solid #2c2c2e;border-radius:50%;display:flex;align-items:center;justify-content:center;font-size:9px;font-weight:800;color:#71717a;cursor:pointer;}
.p:active{background:#1c1c1e;border-color:#ef4444;color:#fff;}
</style>
