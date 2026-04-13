---
title: Statistics for Psychology Research - PSYCH 248, Hunter College, CUNY 2026
summary: ANOVA demo | F-Ratio 
date: 2026-01-01
file: [anova_visual_explainer.html](https://github.com/user-attachments/files/26678985/anova_visual_explainer.html)

<style>
*{box-sizing:border-box}
.ctrl-row{display:flex;gap:20px;flex-wrap:wrap;align-items:flex-end;margin-bottom:1rem}
.ctrl{display:flex;flex-direction:column;gap:4px;flex:1;min-width:120px}
.ctrl label{font-size:12px;color:var(--color-text-secondary)}
.ctrl span{font-size:13px;font-weight:500;color:var(--color-text-primary)}
canvas{display:block;width:100%}
.insight{font-size:13px;color:var(--color-text-secondary);margin:10px 0 0;min-height:36px;line-height:1.6}
.insight b{color:var(--color-text-primary);font-weight:500}
.section-label{font-size:11px;font-weight:500;text-transform:uppercase;letter-spacing:.06em;color:var(--color-text-tertiary);margin:1.5rem 0 .5rem}
.legend{display:flex;gap:14px;flex-wrap:wrap;margin-bottom:10px}
.leg-item{display:flex;align-items:center;gap:5px;font-size:12px;color:var(--color-text-secondary)}
.leg-dot{width:10px;height:10px;border-radius:50%;flex-shrink:0}
.leg-line{width:18px;height:2px;flex-shrink:0}
.divider{border:none;border-top:0.5px solid var(--color-border-tertiary);margin:1.5rem 0}
</style>

<div style="padding:1rem 0">

<div class="ctrl-row">
  <div class="ctrl">
    <label>How far apart are the group means?</label>
    <input type="range" id="sl-between" min="1" max="5" step="1" value="3" oninput="update()">
    <span id="lbl-between">moderate</span>
  </div>
  <div class="ctrl">
    <label>How spread out is each group?</label>
    <input type="range" id="sl-within" min="1" max="5" step="1" value="2" oninput="update()">
    <span id="lbl-within">low</span>
  </div>
  <div class="ctrl">
    <label>People per group</label>
    <input type="range" id="sl-n" min="5" max="40" step="5" value="15" oninput="update()">
    <span id="lbl-n">15</span>
  </div>
</div>

<div class="section-label">Signal vs noise</div>
<div class="legend">
  <span class="leg-item"><span class="leg-dot" style="background:#1D9E75"></span>Group A</span>
  <span class="leg-item"><span class="leg-dot" style="background:#185FA5"></span>Group B</span>
  <span class="leg-item"><span class="leg-dot" style="background:#D85A30"></span>Group C</span>
  <span class="leg-item"><span class="leg-line" style="background:var(--color-text-secondary);opacity:.4"></span>Grand mean</span>
  <span class="leg-item"><span class="leg-line" style="height:3px;background:#1D9E75"></span>Group mean</span>
</div>
<canvas id="dot-canvas" style="height:180px"></canvas>
<div class="insight" id="sig-noise-insight"></div>

<hr class="divider">

<div class="section-label">Degrees of freedom — how N and k shape the test</div>
<canvas id="df-canvas" style="height:90px"></canvas>
<div class="insight" id="df-insight"></div>

<hr class="divider">

<div class="section-label">Tukey's HSD — which pairs actually differ?</div>
<canvas id="tukey-canvas" style="height:120px"></canvas>
<div class="insight" id="tukey-insight"></div>

<hr class="divider">

<div class="section-label">η² — how much of the total spread does group membership explain?</div>
<canvas id="eta-canvas" style="height:60px"></canvas>
<div class="insight" id="eta-insight"></div>

</div>

<script>
const GRP = ['A','B','C'];
const COLORS = ['#1D9E75','#185FA5','#D85A30'];
const BM = [0.5,1.1,1.8,2.8,4.0];
const WM = [0.35,0.7,1.2,2.0,3.2];
const LABELS = ['very low','low','moderate','high','very high'];
const k=3;
let st={between:3,within:2,n:15};
let C={};

function seededRand(seed){
  let s=seed;
  return ()=>{s=(s*1664525+1013904223)&0xffffffff;return(s>>>0)/0xffffffff};
}

function genData(b,w,n){
  const bm=BM[b-1],wm=WM[w-1];
  const means=[-bm*3,0,bm*3].map(x=>x+15);
  const rand=seededRand(b*1000+w*100+n);
  const groups=means.map((mu,gi)=>{
    const pts=[];
    for(let i=0;i<n;i++){let u=0;for(let j=0;j<6;j++)u+=rand();u=(u-3)*wm*3.5;pts.push(mu+u);}
    return pts;
  });
  return{groups,means};
}

function stats(groups,n){
  const N=k*n,all=groups.flat();
  const grand=all.reduce((a,b)=>a+b,0)/N;
  const gm=groups.map(g=>g.reduce((a,b)=>a+b,0)/n);
  const SSb=gm.reduce((s,m)=>s+n*(m-grand)**2,0);
  const SSw=groups.reduce((s,g,gi)=>s+g.reduce((ss,x)=>ss+(x-gm[gi])**2,0),0);
  const SSt=SSb+SSw;
  const dfb=k-1,dfw=N-k;
  const MSb=SSb/dfb,MSw=SSw/dfw;
  const F=MSb/MSw,eta2=SSb/SSt;
  return{gm,grand,SSb,SSw,SSt,dfb,dfw,MSb,MSw,F,eta2,N};
}

function qCrit(dfw){
  const t=[[5,3.64],[8,3.26],[10,3.15],[15,2.98],[20,2.92],[30,2.89],[60,2.83],[999,2.80]];
  for(let i=0;i<t.length;i++)if(dfw<=t[i][0])return t[i][1];
  return 2.80;
}

function isDark(){return window.matchMedia('(prefers-color-scheme:dark)').matches}

function setupCanvas(id){
  const c=document.getElementById(id);
  const dpr=window.devicePixelRatio||1;
  const r=c.getBoundingClientRect();
  c.width=r.width*dpr;c.height=r.height*dpr;
  const ctx=c.getContext('2d');ctx.scale(dpr,dpr);
  return{ctx,W:r.width,H:r.height};
}

function drawDots(){
  const{ctx,W,H}=setupCanvas('dot-canvas');
  const{groups}=genData(st.between,st.within,st.n);
  const s=C;
  const all=groups.flat();
  const minV=Math.min(...all)-3,maxV=Math.max(...all)+3;
  const toX=v=>40+(v-minV)/(maxV-minV)*(W-60);
  const slotH=(H-20)/k;
  const dark=isDark();
  const textColor=dark?'#9c9a92':'#73726c';

  groups.forEach((pts,gi)=>{
    const y=10+gi*slotH+slotH/2;
    pts.forEach((v,i)=>{
      const jitter=((i*7+gi*13)%11-5)*1.5;
      ctx.beginPath();ctx.arc(toX(v),y+jitter,4.5,0,Math.PI*2);
      ctx.fillStyle=COLORS[gi]+'66';ctx.fill();
    });
    ctx.beginPath();ctx.moveTo(toX(s.gm[gi]),y-14);ctx.lineTo(toX(s.gm[gi]),y+14);
    ctx.strokeStyle=COLORS[gi];ctx.lineWidth=3;ctx.stroke();
    ctx.font='12px sans-serif';ctx.fillStyle=textColor;
    ctx.textAlign='left';ctx.fillText(GRP[gi],2,y+4);
  });

  ctx.setLineDash([5,5]);
  ctx.beginPath();ctx.moveTo(toX(s.grand),6);ctx.lineTo(toX(s.grand),H-6);
  ctx.strokeStyle=textColor;ctx.lineWidth=1;ctx.stroke();
  ctx.setLineDash([]);

  const sig=s.F>3;
  document.getElementById('sig-noise-insight').innerHTML=
    sig?`<b>The group means are clearly separated</b> relative to the scatter within each group — the signal is stronger than the noise.`
       :`<b>The group means overlap with the within-group scatter</b> — it's hard to tell the groups apart. The noise drowns out the signal.`;
}

function drawDF(){
  const{ctx,W,H}=setupCanvas('df-canvas');
  const s=C;
  const dark=isDark();
  const textColor=dark?'#9c9a92':'#73726c';
  const primaryColor=dark?'#c2c0b6':'#3d3d3a';
  const pad=16,bH=28,bY=(H-bH)/2;
  const N=s.N;

  const bW=(W-pad*2-20)/N;
  const colors=[];
  for(let gi=0;gi<k;gi++)for(let i=0;i<st.n;i++)colors.push(gi);

  colors.forEach((gi,i)=>{
    const x=pad+i*(bW+1);
    ctx.fillStyle=COLORS[gi]+(gi===0?'cc':'66');
    ctx.fillRect(x,bY,Math.max(bW-1,1),bH);
  });

  ctx.font='11px sans-serif';ctx.fillStyle=textColor;ctx.textAlign='left';
  ctx.fillText(`N = ${N} total`,pad,bY-6);
  ctx.textAlign='right';
  ctx.fillText(`k = ${k} groups`,W-pad,bY-6);

  const dfbW=(k-1)*(bW+1)*st.n/k;
  ctx.strokeStyle='#185FA5';ctx.lineWidth=1.5;
  ctx.beginPath();ctx.moveTo(pad,bY+bH+10);ctx.lineTo(pad+dfbW*2.5,bY+bH+10);ctx.stroke();
  ctx.fillStyle='#185FA5';ctx.textAlign='left';ctx.font='11px sans-serif';
  ctx.fillText(`df between = ${s.dfb}`,pad,bY+bH+22);

  ctx.strokeStyle='#D85A30';ctx.lineWidth=1.5;
  ctx.beginPath();ctx.moveTo(W-pad,bY+bH+10);ctx.lineTo(W-pad-Math.min(s.dfw*(bW+1),W*0.5),bY+bH+10);ctx.stroke();
  ctx.fillStyle='#D85A30';ctx.textAlign='right';
  ctx.fillText(`df within = ${s.dfw}`,W-pad,bY+bH+22);

  document.getElementById('df-insight').innerHTML=
    `With <b>${k} groups</b> and <b>${N} people</b>, the test has <b>${s.dfb} degree${s.dfb>1?'s':''} of freedom between groups</b> and <b>${s.dfw} within groups</b>. More participants → more df within → a more sensitive test.`;
}

function drawTukey(){
  const{ctx,W,H}=setupCanvas('tukey-canvas');
  const s=C;
  const q=qCrit(s.dfw);
  const HSD=q*Math.sqrt(s.MSw/st.n);
  const pairs=[];
  for(let i=0;i<k;i++)for(let j=i+1;j<k;j++)pairs.push({a:i,b:j,diff:Math.abs(s.gm[i]-s.gm[j])});

  const maxDiff=Math.max(...pairs.map(p=>p.diff),HSD)*1.15;
  const pad=16,barH=18,gap=10;
  const labelW=70,barStart=labelW+pad,barEnd=W-pad;
  const barMax=barEnd-barStart;
  const dark=isDark();
  const textColor=dark?'#9c9a92':'#73726c';
  const primaryColor=dark?'#c2c0b6':'#3d3d3a';

  const hsdX=barStart+(HSD/maxDiff)*barMax;
  ctx.beginPath();ctx.moveTo(hsdX,0);ctx.lineTo(hsdX,H-18);
  ctx.setLineDash([4,4]);ctx.strokeStyle=textColor;ctx.lineWidth=1;ctx.stroke();
  ctx.setLineDash([]);
  ctx.font='10px sans-serif';ctx.fillStyle=textColor;ctx.textAlign='center';
  ctx.fillText('HSD threshold',hsdX,H-6);

  pairs.forEach((p,i)=>{
    const y=12+i*(barH+gap);
    const bW=(p.diff/maxDiff)*barMax;
    const sig=p.diff>HSD;
    ctx.fillStyle=sig?COLORS[p.a]+'cc':textColor+'44';
    ctx.fillRect(barStart,y,bW,barH);
    ctx.font='12px sans-serif';ctx.fillStyle=dark?'#c2c0b6':'#3d3d3a';ctx.textAlign='right';
    ctx.fillText(`${GRP[p.a]} vs ${GRP[p.b]}`,barStart-6,y+barH/2+4);
    if(sig){
      ctx.font='11px sans-serif';ctx.fillStyle=COLORS[p.a];ctx.textAlign='left';
      ctx.fillText('*',barStart+bW+5,y+barH/2+4);
    }
  });

  const sigCount=pairs.filter(p=>p.diff>HSD).length;
  document.getElementById('tukey-insight').innerHTML=
    sigCount===0?`No pair of groups is far enough apart to clear the HSD threshold — we can't say which specific groups differ.`
    :sigCount===pairs.length?`<b>All three pairs</b> clear the threshold — every group is significantly different from every other.`
    :`<b>${sigCount} of ${pairs.length} pairs</b> clear the threshold (marked *). The F-test said <em>something</em> differed — Tukey's tells us exactly which pairs.`;
}

function drawEta(){
  const{ctx,W,H}=setupCanvas('eta-canvas');
  const s=C;
  const dark=isDark();
  const textColor=dark?'#9c9a92':'#73726c';
  const pad=16,bH=22,bY=10;
  const bW=W-pad*2;

  ctx.fillStyle=dark?'#2c2c2a':'#f1efe8';
  ctx.beginPath();ctx.roundRect(pad,bY,bW,bH,4);ctx.fill();

  const fillW=bW*Math.min(s.eta2,1);
  ctx.fillStyle='#1D9E75cc';
  ctx.beginPath();ctx.roundRect(pad,bY,fillW,bH,4);ctx.fill();

  const markers=[{v:.01,l:'small'},{v:.06,l:'medium'},{v:.14,l:'large'}];
  markers.forEach(m=>{
    const x=pad+m.v*bW;
    ctx.beginPath();ctx.moveTo(x,bY+bH+2);ctx.lineTo(x,bY+bH+8);
    ctx.strokeStyle=textColor;ctx.lineWidth=1;ctx.stroke();
    ctx.font='10px sans-serif';ctx.fillStyle=textColor;ctx.textAlign='center';
    ctx.fillText(m.l,x,bY+bH+20);
  });

  ctx.font='11px sans-serif';ctx.fillStyle='#fff';ctx.textAlign='left';
  if(fillW>50)ctx.fillText((s.eta2*100).toFixed(0)+'%',pad+6,bY+bH/2+4);

  const interp=s.eta2<.01?'negligible':s.eta2<.06?'small':s.eta2<.14?'medium':'large';
  document.getElementById('eta-insight').innerHTML=
    `Group membership explains <b>${(s.eta2*100).toFixed(0)}% of the total variance</b> in scores — a <b>${interp} effect</b> by convention.`;
}

function update(){
  st.between=+document.getElementById('sl-between').value;
  st.within=+document.getElementById('sl-within').value;
  st.n=+document.getElementById('sl-n').value;
  document.getElementById('lbl-between').textContent=LABELS[st.between-1];
  document.getElementById('lbl-within').textContent=LABELS[st.within-1];
  document.getElementById('lbl-n').textContent=st.n;

  const{groups}=genData(st.between,st.within,st.n);
  C=stats(groups,st.n);
  drawDots();drawDF();drawTukey();drawEta();
}

window.addEventListener('resize',()=>update());
update();
</script>


---
