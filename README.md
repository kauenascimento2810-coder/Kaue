
<!DOCTYPE html>
<html lang="pt-BR">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1, maximum-scale=1, user-scalable=no">
<title>NITRO RUSH HD</title>
<style>
*{box-sizing:border-box;touch-action:manipulation}
body{margin:0;background:#080d18;color:white;font-family:Arial;text-align:center;overflow:hidden}
header{padding:10px;background:linear-gradient(90deg,#e11d48,#f97316);font-size:23px;font-weight:900}
#info{display:flex;justify-content:space-around;padding:8px;background:#111827;font-weight:bold}
canvas{display:block;width:100%;height:calc(100dvh - 190px);background:#287442}
.controls{display:flex;justify-content:space-around;gap:8px;padding:10px}
button{border:2px solid #64748b;border-radius:14px;padding:15px 20px;background:#1e293b;color:white;font-weight:bold;font-size:18px}
#nitro{background:#dc2626}
#overlay{position:fixed;inset:0;background:#020617dd;display:flex;align-items:center;justify-content:center;z-index:3}
.panel{background:#111827;border:2px solid #f97316;border-radius:20px;padding:25px;width:85%;max-width:350px}
.panel h1{color:#fb923c}
.panel button{background:#ea580c;margin:7px}
.hidden{display:none!important}
</style>
</head>
<body>
<header>🏁 NITRO RUSH HD</header>
<div id="info">
<span>🏆 <span id="score">0</span></span>
<span>⚡ <span id="speed">0</span> km/h</span>
<span>🔥 <span id="fuel">100</span>%</span>
</div>
<canvas id="game"></canvas>
<div class="controls">
<button id="left">⬅️</button>
<button id="nitro">🔥 NITRO</button>
<button id="right">➡️</button>
</div>
<div id="overlay">
<div class="panel">
<h1>🏎️ NITRO RUSH</h1>
<p>Prepare-se para a corrida!</p>
<p>Desvie dos carros e use o nitro!</p>
<button id="play">▶ JOGAR</button>
</div>
</div>
<script>
const canvas=document.getElementById("game");
const ctx=canvas.getContext("2d");
const overlay=document.getElementById("overlay");
let W=0,H=0;
function resize(){
 const d=Math.min(devicePixelRatio||1,2);
 W=innerWidth;H=innerHeight-190;
 canvas.width=W*d;canvas.height=Math.max(H,200)*d;
 canvas.style.height=Math.max(H,200)+"px";
 ctx.setTransform(d,0,0,d,0,0);
 H=Math.max(H,200);
}
addEventListener("resize",resize);resize();

let running=false,playerX=0,roadOffset=0,score=0;
let speed=260,fuel=100,boost=false,last=0,traffic=[];
let left=false,right=false;
const clamp=(v,a,b)=>Math.max(a,Math.min(b,v));

function car(x,y,w,h,color,enemy=false){
 ctx.fillStyle="#111";
 ctx.fillRect(x-w*.53,y+h*.15,w*.18,h*.27);
 ctx.fillRect(x+w*.35,y+h*.15,w*.18,h*.27);
 ctx.fillRect(x-w*.53,y+h*.66,w*.18,h*.24);
 ctx.fillRect(x+w*.35,y+h*.66,w*.18,h*.24);
 ctx.fillStyle=color;
 ctx.beginPath();
 ctx.moveTo(x-w*.38,y+h);
 ctx.lineTo(x-w*.48,y+h*.35);
 ctx.lineTo(x-w*.28,y+h*.08);
 ctx.lineTo(x+w*.28,y+h*.08);
 ctx.lineTo(x+w*.48,y+h*.35);
 ctx.lineTo(x+w*.38,y+h);
 ctx.closePath();ctx.fill();
 ctx.fillStyle="#172554";
 ctx.fillRect(x-w*.27,y+h*.25,w*.54,h*.22);
 ctx.fillStyle="#93c5fd";
 ctx.fillRect(x-w*.22,y+h*.28,w*.44,h*.08);
 ctx.fillStyle="#fef3c7";
 ctx.fillRect(x-w*.32,y+h*.86,w*.18,h*.07);
 ctx.fillRect(x+w*.14,y+h*.86,w*.18,h*.07);
 if(!enemy&&boost&&running){
  ctx.fillStyle="#fb923c";
  ctx.beginPath();
  ctx.moveTo(x-w*.2,y+h);ctx.lineTo(x,y+h+18+Math.random()*10);
  ctx.lineTo(x+w*.2,y+h);ctx.fill();
 }
}

function road(){
 ctx.fillStyle="#277444";ctx.fillRect(0,0,W,H);
 for(let i=0;i<12;i++){
  let y=(i*100+roadOffset*1.8)%(H+100)-50;
  ctx.fillStyle="#338a4d";
  ctx.fillRect(0,y, W,12);
 }
 let bend=Math.sin(score/450)*W*.13;
 let cx=W/2+bend;
 ctx.fillStyle="#cbd5e1";
 ctx.beginPath();
 ctx.moveTo(cx-W*.39,-20);ctx.lineTo(cx+W*.39,-20);
 ctx.lineTo(cx+W*.49,H);ctx.lineTo(cx-W*.49,H);ctx.closePath();ctx.fill();
 ctx.fillStyle="#30343b";
 ctx.beginPath();
 ctx.moveTo(cx-W*.37,-20);ctx.lineTo(cx+W*.37,-20);
 ctx.lineTo(cx+W*.46,H);ctx.lineTo(cx-W*.46,H);ctx.closePath();ctx.fill();
 for(let i=0;i<16;i++){
  let y=(i*65+roadOffset*1.8)%(H+65)-30;
  let x=cx+(y/H-.5)*bend*.25;
  ctx.fillStyle="#fff";
  ctx.fillRect(x-W*.125,y,5,30);
  ctx.fillRect(x+W*.125,y,5,30);
 }
 return cx;
}

function reset(){
 score=0;speed=260;fuel=100;roadOffset=0;
 playerX=W/2;traffic=[];
 for(let i=0;i<3;i++)spawn(-200-i*260);
 running=true;last=performance.now();
 overlay.classList.add("hidden");
 requestAnimationFrame(loop);
}
function spawn(y=-100){
 traffic.push({
  x:W*.32+Math.random()*W*.36,
  y:y,color:["#e11d48","#facc15","#38bdf8","#a855f7"][Math.floor(Math.random()*4)]
 });
}
function loop(now){
 if(!running)return;
 const dt=Math.min((now-last)/1000,.04);last=now;
 const cx=road();
 if(left)playerX-=W*.75*dt;
 if(right)playerX+=W*.75*dt;
 playerX=clamp(playerX,W*.12,W*.88);
 speed=boost&&fuel>0?420:260;
 if(boost&&fuel>0)fuel=Math.max(0,fuel-32*dt);
 else fuel=Math.min(100,fuel+8*dt);
 roadOffset+=speed*dt;
 score+=speed*dt/10;
 for(let t of traffic){
  t.y+=(speed*.75+80)*dt;
  let bend=Math.sin(score/450)*W*.13;
  t.x=clamp(t.x+bend*.0008, W*.25,W*.75);
  if(Math.abs(t.x-playerX)<W*.075&&
     Math.abs(t.y-(H*.70))<45){
   running=false;
   document.querySelector(".panel").innerHTML=
    "<h1>💥 FIM DE JOGO</h1><p>Pontuação: "+Math.floor(score)+
    "</p><button onclick='reset()'>🔄 JOGAR DE NOVO</button>";
   overlay.classList.remove("hidden");return;
  }
 }
 traffic=traffic.filter(t=>t.y<H+100);
 if(traffic.length<4&&Math.random()<.025)spawn(-100);
 for(const t of traffic)car(t.x,t.y,W*.095,65,t.color,true);
 car(playerX,H*.70,W*.115,85,"#ef2929");
 document.getElementById("score").textContent=Math.floor(score);
 document.getElementById("speed").textContent=speed===420?420:260;
 document.getElementById("fuel").textContent=Math.floor(fuel);
 requestAnimationFrame(loop);
}
function hold(id,down,up){
 const b=document.getElementById(id);
 b.addEventListener("pointerdown",e=>{e.preventDefault();down()});
 b.addEventListener("pointerup",e=>{e.preventDefault();up()});
 b.addEventListener("pointerleave",up);
 b.addEventListener("pointercancel",up);
}
hold("left",()=>left=true,()=>left=false);
hold("right",()=>right=true,()=>right=false);
hold("nitro",()=>boost=true,()=>boost=false);
document.getElementById("play").onclick=reset;
</script>
</body>
</html>
