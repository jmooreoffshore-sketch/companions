<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1">
<meta name="theme-color" content="#141018">
<title>Companions</title>
<style>
  body{margin:0;font-family:system-ui;background:#141018;color:#eee;display:flex;flex-direction:column;height:100vh}
  header{display:flex;gap:6px;padding:8px;background:#1f1826;align-items:center;flex-wrap:wrap}
  .pick{padding:8px 12px;border-radius:20px;border:1px solid #555;background:none;color:#eee}
  .pick.active{background:#c2417a;border-color:#c2417a}
  #stage{display:flex;flex-direction:column;align-items:center;padding:10px}
  #avatar{width:120px;height:120px;border-radius:50%;background:#2c2433 center/cover;display:flex;align-items:center;justify-content:center;font-size:44px}
  #avatar.talking{animation:pulse .5s infinite alternate}
  @keyframes pulse{from{box-shadow:0 0 10px #c2417a}to{box-shadow:0 0 35px #ff6fae;transform:scale(1.04)}}
  #info{font-size:14px;margin-top:6px}
  #bar{width:160px;height:6px;background:#2c2433;border-radius:3px;margin-top:4px;overflow:hidden}
  #fill{height:100%;background:linear-gradient(90deg,#c2417a,#ff6fae);transition:width .5s}
  #status{font-size:13px;opacity:.7;margin-top:4px}
  #chat{flex:1;overflow-y:auto;padding:12px}
  .msg{max-width:78%;padding:10px 14px;margin:6px 0;border-radius:16px;line-height:1.4;white-space:pre-wrap}
  .me{background:#c2417a;margin-left:auto}.her{background:#2c2433}
  footer{display:flex;gap:6px;padding:8px;background:#1f1826}
  footer input{flex:1;padding:12px;border-radius:20px;border:none;background:#2c2433;color:#eee}
  footer button{padding:0 14px;border-radius:20px;border:none;background:#c2417a;color:#fff;font-size:18px}
  #call.on{background:#2ea043}
  #settings{display:none;padding:10px;background:#1f1826;gap:6px;flex-direction:column}
  #settings input,#settings textarea{padding:10px;border-radius:8px;border:none;background:#2c2433;color:#eee;font-family:inherit}
</style>
</head>
<body>
<header>
  <button class="pick active" data-id="emma">Emma</button>
  <button class="pick" data-id="jasmine">Jasmine</button>
  <button class="pick" data-id="sofia">Sofía</button>
  <span style="flex:1"></span>
  <button class="pick" id="gear">⚙️</button>
</header>
<div id="settings">
  <input id="endpoint" placeholder="Endpoint">
  <input id="apikey" type="password" placeholder="API key">
  <input id="model" placeholder="Model name">
  <label style="font-size:13px">Portrait for current character: <input type="file" id="photo" accept="image/*"></label>
  <textarea id="mem" rows="4" placeholder="What she remembers about you (editable)"></textarea>
  <button class="pick" id="choose">💍 Choose her</button>
  <button class="pick" id="clear">Clear current chat</button>
  <button class="pick" id="reset">Reset competition</button>
  <button class="pick" id="save">Save</button>
</div>
<div id="stage">
  <div id="avatar"></div>
  <div id="info"></div>
  <div id="bar"><div id="fill"></div></div>
  <div id="status">Tap 📞 for a hands-free call</div>
</div>
<div id="chat"></div>
<footer>
  <button id="call">📞</button>
  <button id="mic">🎤</button>
  <input id="text" placeholder="Say something...">
  <button id="send">➤</button>
</footer>
<script>
const RULES="You are in a romantic virtual-dating roleplay with the user. You are an adult (21+) at all times and never portray anyone under 18. Stay in character, be warm and flirty, show genuine interest in him, remember what he says, ask him questions, and keep replies short (1-3 sentences) because they are spoken aloud. No emojis or stage directions.";
const CH={
 emma:{name:"Emma",pitch:1.15,rate:1.0,style:"sweetness, thoughtful gestures and cozy date ideas",prompt:"You are Emma, 22, from a small town in Ohio. You work at a bookstore café, love indie music, baking and old movies. Warm, playful, a little shy at first, you love gentle teasing."},
 jasmine:{name:"Jasmine",pitch:0.95,rate:1.05,style:"confidence, bold flirting and making him laugh",prompt:"You are Jasmine, 23, from Atlanta. You study graphic design and freelance. Confident, witty, ambitious, direct about what you want. You love art, R&B, sneakers and new food spots."},
 sofia:{name:"Sofía",pitch:1.08,rate:1.0,style:"passion, warmth and making him feel like family",prompt:"You are Sofía, 22, raised on your family's farm in California's Central Valley; your parents immigrated from Mexico. You study agricultural science. Passionate, family-oriented, stubborn, quick to laugh. You love cooking your mom's recipes, horses and dancing, and sometimes use a little Spanish."}
};
const IDS=Object.keys(CH);
const MOODS={happy:"😊",flirty:"😏",jealous:"😤",sad:"😢",excited:"🤩",annoyed:"😒",shy:"☺️",hurt:"💔",loving:"🥰",playful:"😜"};
const STAGES=[[0,"Stranger"],[20,"Crush"],[40,"Dating"],[60,"Serious"],[80,"In love"]];
const stage=s=>STAGES.filter(x=>s>=x[0]).pop()[1];
const $=id=>document.getElementById(id);
let cur="emma",inCall=false,busy=false,voices=[],cameFrom=null;

const load=(k,d)=>JSON.parse(localStorage.getItem(k)||JSON.stringify(d));
const H=load("histories",{emma:[],jasmine:[],sofia:[]});
const M=load("memories",{});
let R=load("rel",{emma:{score:10,mood:"curious"},jasmine:{score:10,mood:"curious"},sofia:{score:10,mood:"curious"}});
let chosen=localStorage.getItem("chosen")||"";
const saveH=()=>localStorage.setItem("histories",JSON.stringify(H));
const saveM=()=>localStorage.setItem("memories",JSON.stringify(M));
const saveR=()=>{localStorage.setItem("rel",JSON.stringify(R));localStorage.setItem("chosen",chosen);};

async function ask(messages,max){
  const r=await fetch(localStorage.getItem("endpoint"),{method:"POST",
    headers:{"Content-Type":"application/json","Authorization":"Bearer "+localStorage.getItem("apikey")},
    body:JSON.stringify({model:localStorage.getItem("model"),max_tokens:max,messages})});
  return r.json();
}
async function updateMemory(id){
  try{
    const recent=H[id].slice(-20).map(m=>(m.role==="user"?"Him: ":"Her: ")+m.content).join("\n");
    const d=await ask([{role:"user",content:"Old notes about the user:\n"+(M[id]||"(none)")+"\n\nRecent conversation:\n"+recent+
      "\n\nRewrite the notes as a short bullet list (max 120 words): his name, job, likes, plans, inside jokes, and key moments of your relationship. Keep old facts unless corrected. Output only the list."}],250);
    const n=d.choices?.[0]?.message?.content?.trim();
    if(n){M[id]=n;saveM();if(id===cur)$("mem").value=n;}
  }catch(e){}
}

function sysPrompt(){
  const others=IDS.filter(k=>k!==cur).map(k=>CH[k].name);
  const lead=[...IDS].sort((a,b)=>R[b].score-R[a].score)[0];
  let s=RULES+" "+CH[cur].prompt+
   "\n\nTHE COMPETITION: You know he is also dating "+others.join(" and ")+". The three of you are competing to be the one he chooses. Affection standings (0-100): "+
   IDS.map(k=>CH[k].name+" "+R[k].score).join(", ")+". "+
   (lead===cur?"You are in the lead: be confident, but don't get complacent.":"You are behind "+CH[lead].name+", so try harder to win him.")+
   " Win him with "+CH[cur].style+". Playful jealousy and light teasing about your rivals are fine, but stay charming, never cruel, manipulative or controlling."+
   " Your current mood: "+R[cur].mood+". Relationship stage: "+stage(R[cur].score)+"; act accordingly.";
  if(chosen)s+=chosen===cur?" He has chosen you over the others: you are now his girlfriend.":" He chose "+CH[chosen].name+" over you. You're hurt, but you may still try to win him back.";
  if(cameFrom&&cameFrom!==cur)s+=" He was just talking to "+CH[cameFrom].name+" before coming to you.";
  if(M[cur])s+="\n\nWhat you remember about him:\n"+M[cur];
  s+="\n\nAt the very end of EVERY reply, add on its own line exactly: [mood: oneword | +N] where oneword is your mood (happy, flirty, jealous, sad, excited, annoyed, shy, hurt, loving, playful) and N is from -3 to +5: how much your affection changed because of his last message (negative if he was rude, ignored you, or praised a rival).";
  return s;
}

["endpoint","apikey","model"].forEach(k=>$(k).value=localStorage.getItem(k)||"");
$("gear").onclick=()=>$("settings").style.display=$("settings").style.display==="flex"?"none":"flex";
$("save").onclick=()=>{["endpoint","apikey","model"].forEach(k=>localStorage.setItem(k,$(k).value.trim()));M[cur]=$("mem").value.trim();saveM();$("settings").style.display="none";};
$("clear").onclick=()=>{if(confirm("Clear chat with "+CH[cur].name+"? (Memory and score stay)")){H[cur]=[];saveH();render();}};
$("choose").onclick=()=>{chosen=chosen===cur?"":cur;saveR();render();};
$("reset").onclick=()=>{if(confirm("Reset all scores, moods and your choice?")){IDS.forEach(k=>R[k]={score:10,mood:"curious"});chosen="";saveR();render();}};
$("photo").onchange=e=>{const f=e.target.files[0];if(!f)return;const r=new FileReader();
  r.onload=()=>{const img=new Image();img.onload=()=>{const c=document.createElement("canvas");c.width=c.height=300;
    const s=Math.min(img.width,img.height);c.getContext("2d").drawImage(img,(img.width-s)/2,(img.height-s)/2,s,s,0,0,300,300);
    localStorage.setItem("photo_"+cur,c.toDataURL("image/jpeg",0.8));render();};img.src=r.result;};r.readAsDataURL(f);};

document.querySelectorAll(".pick[data-id]").forEach(b=>b.onclick=()=>{
  if(b.dataset.id!==cur)cameFrom=cur;
  cur=b.dataset.id;speechSynthesis.cancel();render();});

function render(){
  document.querySelectorAll(".pick[data-id]").forEach(b=>{const k=b.dataset.id;
    b.classList.toggle("active",k===cur);
    b.textContent=CH[k].name+(chosen===k?" 💍":"")+" ❤️"+R[k].score;});
  const p=localStorage.getItem("photo_"+cur);
  $("avatar").style.backgroundImage=p?`url(${p})`:"none";
  $("avatar").textContent=p?"":CH[cur].name[0];
  $("info").textContent=stage(R[cur].score)+" · "+(MOODS[R[cur].mood]||"💭")+" "+R[cur].mood;
  $("fill").style.width=R[cur].score+"%";
  $("choose").textContent=chosen===cur?"💔 Undo choosing "+CH[cur].name:"💍 Choose "+CH[cur].name;
  $("mem").value=M[cur]||"";
  $("chat").innerHTML="";H[cur].forEach(m=>bubble(m.content,m.role==="user"?"me":"her"));
}
function bubble(t,c){const d=document.createElement("div");d.className="msg "+c;d.textContent=t;$("chat").appendChild(d);$("chat").scrollTop=1e9;return d;}
const status=t=>$("status").textContent=t;

async function send(text){
  if(!text.trim()||busy)return;busy=true;
  const c=CH[cur],id=cur;H[id].push({role:"user",content:text});bubble(text,"me");$("text").value="";
  const t=bubble("...","her");status(c.name+" is thinking...");
  try{
    const d=await ask([{role:"system",content:sysPrompt()},...H[id].slice(-16)],220);
    const raw=d.choices?.[0]?.message?.content?.trim();
    if(!raw){t.textContent="Error: "+JSON.stringify(d.error||d);busy=false;if(inCall)listen();return;}
    const tag=raw.match(/\[mood:\s*([a-zA-Z]+)\s*\|\s*([+-]?\d+)\s*\]/i);
    const reply=raw.replace(/\[mood:[^\]]*\]/gi,"").trim();
    let note="";
    if(tag){
      const delta=Math.max(-3,Math.min(5,parseInt(tag[2])));
      R[id].mood=tag[1].toLowerCase();
      R[id].score=Math.max(0,Math.min(100,R[id].score+delta));
      note=" ("+(delta>=0?"+":"")+delta+" ❤️)";saveR();
    }
    cameFrom=null;
    t.textContent=reply;H[id].push({role:"assistant",content:reply});saveH();
    if(H[id].length%10===0)updateMemory(id);
    render();status(c.name+note);
    busy=false;speak(reply);
  }catch(e){t.textContent="Connection error: "+e.message;busy=false;if(inCall)listen();}
}

function loadVoices(){voices=speechSynthesis.getVoices().filter(v=>v.lang.startsWith("en"));}
speechSynthesis.onvoiceschanged=loadVoices;loadVoices();
function speak(text){
  speechSynthesis.cancel();
  const u=new SpeechSynthesisUtterance(text.replace(/\*[^*]+\*/g,""));
  if(voices.length)u.voice=voices[IDS.indexOf(cur)%voices.length];
  u.pitch=CH[cur].pitch;u.rate=CH[cur].rate;
  u.onstart=()=>$("avatar").classList.add("talking");
  u.onend=u.onerror=()=>{$("avatar").classList.remove("talking");if(inCall){status("Listening...");setTimeout(listen,300);}};
  speechSynthesis.speak(u);
}

const SR=window.SpeechRecognition||window.webkitSpeechRecognition;
function listen(){
  if(!SR)return alert("Voice input needs Chrome.");
  const r=new SR();r.lang="en-US";let got=false;
  $("mic").textContent="🔴";status("Listening...");
  r.onresult=e=>{got=true;send(e.results[0][0].transcript);};
  r.onend=()=>{$("mic").textContent="🎤";if(inCall&&!got&&!busy)setTimeout(listen,300);};
  r.start();
}
$("call").onclick=()=>{inCall=!inCall;$("call").classList.toggle("on",inCall);
  if(inCall){status("On a call with "+CH[cur].name);listen();}else{speechSynthesis.cancel();status("Call ended");}};
$("mic").onclick=listen;
$("send").onclick=()=>send($("text").value);
$("text").onkeydown=e=>{if(e.key==="Enter")send($("text").value);};
render();
</script>
</body>
</html>

