# Chess-terry-davis-
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<title>Chess</title>
<style>
  html,body{margin:0;background:#000;height:100%}
  body{display:flex;flex-direction:column;align-items:center;font-family:"Courier New",monospace;font-weight:bold}
  #wrap{width:100%;max-width:960px;background:#ffff55;min-height:100%;display:flex;flex-direction:column}
  #bar{background:#0000aa;color:#fff;padding:3px 8px;display:flex;justify-content:space-between;font-size:14px;white-space:nowrap}
  svg{width:100%;height:auto;display:block;user-select:none;-webkit-tap-highlight-color:transparent}
  #pc,#hl,#lb{pointer-events:none}
  [data-s]{cursor:pointer}
  #ctl{display:flex;gap:12px;justify-content:center;padding:6px 0 18px}
  button{font:inherit;background:#fff;color:#000;border:2px solid #000;padding:8px 18px;border-radius:0;cursor:pointer}
  button:active{background:#0000aa;color:#fff}
  button:focus-visible{outline:3px solid #0000aa;outline-offset:2px}
</style>
</head>
<body>
<div id="wrap">
  <div id="bar"><span id="clk"></span><span id="st">Loading engine…</span></div>
  <svg id="board" viewBox="0 0 800 600" role="img" aria-label="Chess board">
    <defs>
      <g id="p"><circle cx="20" cy="16" r="7"/><path d="M14 47C15 36 17 28 17 23H23C23 28 25 36 26 47Z"/><rect x="10" y="46" width="20" height="9" rx="2"/></g>
      <g id="r"><path d="M10 6H15V11H18V6H22V11H25V6H30V22H10Z"/><path d="M13 22H27L26 46H14Z"/><rect x="8" y="46" width="24" height="9" rx="2"/></g>
      <g id="n"><path d="M11 47L13 36L17 30L9 27L7 22L11 16L16 10L18 4L22 8L26 11C32 18 32 32 30 47Z"/><circle cx="16" cy="15" r="1.4" fill="#000" stroke="none"/><rect x="8" y="46" width="24" height="9" rx="2"/></g>
      <g id="b"><circle cx="20" cy="6" r="3"/><path d="M20 9C28 15 28 22 25 27H15C12 22 12 15 20 9Z"/><path d="M18 14L23 19" fill="none" stroke="#000"/><rect x="12" y="26" width="16" height="4" rx="1.5"/><path d="M15 30H25L27 46H13Z"/><rect x="9" y="46" width="22" height="9" rx="2"/></g>
      <g id="q"><path d="M9 12L13 30L16 14L20 30L24 14L27 30L31 12L29 36H11Z"/><circle cx="9" cy="10" r="2.5"/><circle cx="16" cy="11" r="2.5"/><circle cx="24" cy="11" r="2.5"/><circle cx="31" cy="10" r="2.5"/><path d="M12 36H28L29 46H11Z"/><rect x="8" y="46" width="24" height="9" rx="2"/></g>
      <g id="k"><rect x="18.5" y="1" width="3" height="12"/><rect x="15" y="4.5" width="10" height="3"/><path d="M12 22C12 14 28 14 28 22L29 34H11Z"/><path d="M11 34H29L31 46H9Z"/><rect x="8" y="46" width="24" height="9" rx="2"/></g>
    </defs>
    <g id="bd"></g><g id="lb"></g><g id="hl"></g><g id="pc"></g>
  </svg>
  <div id="ctl"><button id="nw">New game</button><button id="un">Undo</button></div>
</div>

<script src="https://cdnjs.cloudflare.com/ajax/libs/chess.js/0.10.3/chess.min.js"></script>
<script>
const THINK_MS = 3000;           // how long Stockfish thinks per move (bigger = stronger)
const SF_URLS = [
  "https://cdnjs.cloudflare.com/ajax/libs/stockfish.js/10.0.2/stockfish.js",
  "https://cdn.jsdelivr.net/npm/stockfish@10.0.2/src/stockfish.js",
  "https://cdn.jsdelivr.net/gh/nmrugg/stockfish.js@10.0.2/src/stockfish.js"
];
const W=44,H=31,X0=92,Y0=290,S=0.8,NS="http://www.w3.org/2000/svg";
const $=id=>document.getElementById(id);
const C=(f,r)=>[X0+W*(f+r),Y0+H*(f-r)];
const nm=(f,r)=>"abcdefgh"[f]+(r+1);
const game=new Chess();
let sel=null,targets=[],last=[],busy=false,awaiting=false,sf=null,tm=0;

/* ---------- board ---------- */
function buildBoard(){
  let s=`<polygon points="${X0-W},${Y0} ${X0+7*W},${Y0+8*H} ${X0+7*W},${Y0+8*H+10} ${X0-W},${Y0+10}" fill="#aaa" stroke="#000" stroke-width="2"/>
  <polygon points="${X0+7*W},${Y0+8*H} ${X0+15*W},${Y0} ${X0+15*W},${Y0+10} ${X0+7*W},${Y0+8*H+10}" fill="#555" stroke="#000" stroke-width="2"/>`;
  for(let f=0;f<8;f++)for(let r=0;r<8;r++){
    const [x,y]=C(f,r);
    s+=`<polygon data-s="${nm(f,r)}" points="${x-W},${y} ${x},${y-H} ${x+W},${y} ${x},${y+H}" fill="${(f+r)%2?"#fff":"#555"}"/>`;
  }
  s+=`<polygon points="${X0},${Y0-H} ${X0+7*W},${Y0-8*H} ${X0+15*W},${Y0} ${X0+7*W},${Y0+8*H} ${X0-W},${Y0}" fill="none" stroke="#000" stroke-width="2.5" pointer-events="none"/>`;
  $("bd").innerHTML=s;
  let t="";
  const L=(x,y,c)=>t+=`<text x="${x}" y="${y}" text-anchor="middle" dominant-baseline="central" font-size="17" font-family="Courier New,monospace" font-weight="bold" fill="#000">${c}</text>`;
  for(let i=0;i<8;i++){
    let [x,y]=C(0,i);L(x-31,y-21,i+1);
    [x,y]=C(i,7);L(x+40,y-15,"ABCDEFGH"[i]);
    [x,y]=C(i,0);L(x-34,y+37,"ABCDEFGH"[i]);
    [x,y]=C(7,i);L(x+36,y+38,i+1);
  }
  $("lb").innerHTML=t;
}
function draw(){
  const list=[];
  game.board().forEach((row,i)=>row.forEach((p,f)=>{if(p)list.push({f,r:7-i,p})}));
  list.sort((a,b)=>(a.f-a.r)-(b.f-b.r));
  $("pc").innerHTML=list.map(({f,r,p})=>{
    const [x,y]=C(f,r);
    return `<use href="#${p.type}" transform="translate(${x-20*S},${y+17-56*S}) scale(${S})" fill="${p.color==="w"?"#fff":"#8c8c8c"}" stroke="#000" stroke-width="${1.7/S}" stroke-linejoin="round"/>`;
  }).join("");
  const poly=(q,a)=>{const f="abcdefgh".indexOf(q[0]),r=+q[1]-1,[x,y]=C(f,r);
    return `<polygon points="${x-W},${y} ${x},${y-H} ${x+W},${y} ${x},${y+H}" ${a}/>`};
  let h="";
  if(last.length)last.forEach(q=>h+=poly(q,'fill="rgba(0,0,170,.28)"'));
  if(game.in_check()){const t=game.turn();game.board().forEach((row,i)=>row.forEach((p,f)=>{if(p&&p.type==="k"&&p.color===t)h+=poly(nm(f,7-i),'fill="rgba(255,0,0,.5)"')}))}
  targets.forEach(m=>h+=poly(m.to,'fill="rgba(255,0,0,.15)" stroke="#d00" stroke-width="2"'));
  if(sel)h+=poly(sel,'fill="none" stroke="#d00" stroke-width="3.5"');
  $("hl").innerHTML=h;
  showStatus();
}
function showStatus(){
  let s;
  if(game.in_checkmate())s=game.turn()==="w"?"Checkmate: Black wins":"Checkmate: you win";
  else if(game.game_over())s="Draw";
  else if(busy)s="Black is thinking…";
  else s=game.in_check()?"Check! Your move":"Your move (White)";
  $("st").textContent=s;
}

/* ---------- play ---------- */
$("bd").addEventListener("click",e=>{
  const q=e.target.getAttribute("data-s");
  if(!q||busy||game.game_over()||game.turn()!=="w")return;
  if(sel&&targets.some(m=>m.to===q)){
    game.move({from:sel,to:q,promotion:"q"});
    last=[sel,q];sel=null;targets=[];draw();
    if(!game.game_over())setTimeout(askEngine,60);
    return;
  }
  const p=game.get(q);
  if(p&&p.color==="w"){sel=q;targets=game.moves({square:q,verbose:true})}
  else{sel=null;targets=[]}
  draw();
});
function playBlack(mv){
  const m=game.move({from:mv.slice(0,2),to:mv.slice(2,4),promotion:mv[4]||"q"});
  if(!m)return false;
  last=[m.from,m.to];return true;
}
function finish(){busy=false;awaiting=false;clearTimeout(tm);draw()}
function askEngine(){
  busy=true;awaiting=true;draw();
  if(sf){
    sf.postMessage("position fen "+game.fen());
    sf.postMessage("go movetime "+THINK_MS);
    tm=setTimeout(()=>{if(awaiting)backup()},THINK_MS+20000);
  }else setTimeout(backup,50);
}
function onEngine(e){
  const l=typeof e.data==="string"?e.data:"";
  if(l==="uciok"){sf.postMessage("setoption name Skill Level value 20");sf.postMessage("isready")}
  else if(l==="readyok"&&!busy){$("st").textContent="Your move (White)"}
  else if(l.startsWith("bestmove")&&awaiting){
    if(!playBlack(l.split(" ")[1]))return backup();
    finish();
  }
}

/* ---------- weak backup engine (only used if Stockfish fails to load) ---------- */
const V={p:100,n:320,b:330,r:500,q:900,k:0};
function ev(){let s=0;game.board().forEach(r=>r.forEach(c=>{if(c)s+=(c.color==="w"?1:-1)*V[c.type]}));return s}
function nega(d,a,b){
  if(d===0)return(game.turn()==="w"?1:-1)*ev();
  const ms=game.moves();if(!ms.length)return game.in_check()?-99999:0;
  let best=-1e9;
  for(const m of ms){game.move(m);const s=-nega(d-1,-b,-a);game.undo();if(s>best)best=s;if(best>a)a=best;if(a>=b)break}
  return best;
}
function backup(){
  awaiting=false;
  if(game.game_over()){finish();return}
  let best=null,bs=-1e9;
  for(const m of game.moves()){game.move(m);const s=-nega(2,-1e9,1e9);game.undo();if(s>bs){bs=s;best=m}}
  const m=game.move(best);last=[m.from,m.to];
  busy=false;draw();$("st").textContent="Backup engine (weak). Check internet.";
}

/* ---------- engine setup ---------- */
function startEngine(){
  try{
    const code="for(const u of "+JSON.stringify(SF_URLS)+"){try{importScripts(u);break}catch(e){}}";
    sf=new Worker(URL.createObjectURL(new Blob([code],{type:"application/javascript"})));
    sf.onmessage=onEngine;
    sf.onerror=()=>{sf=null;$("st").textContent="Stockfish failed to load"};
    sf.postMessage("uci");
  }catch(e){sf=null}
}

/* ---------- buttons ---------- */
$("nw").onclick=()=>{
  if(sf&&awaiting)sf.postMessage("stop");
  awaiting=false;busy=false;clearTimeout(tm);
  game.reset();sel=null;targets=[];last=[];
  if(sf)sf.postMessage("ucinewgame");
  draw();
};
$("un").onclick=()=>{
  if(busy||!game.history().length)return;
  game.undo();if(game.turn()==="b")game.undo();
  sel=null;targets=[];last=[];draw();
};

/* ---------- clock ---------- */
function tick(){
  const d=new Date(),p=n=>String(n).padStart(2,"0");
  $("clk").textContent=d.toLocaleDateString("en-US",{weekday:"short"})+" "+p(d.getMonth()+1)+"/"+p(d.getDate())+" "+p(d.getHours())+":"+p(d.getMinutes())+":"+p(d.getSeconds());
}
setInterval(tick,1000);tick();

buildBoard();draw();startEngine();
</script>
</body>
</html>
 
