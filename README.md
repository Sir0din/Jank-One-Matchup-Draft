<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8"/>
<meta name="viewport" content="width=device-width,initial-scale=1"/>
<title>Jank One – Roster + Gods Draft</title>
<style>
:root{--bg:#0f0f16;--fg:#eef0f7;--muted:#1a1a2b;--accent:#D95E29;--gold:#D4AF37}
*{box-sizing:border-box}
body{margin:0;padding:24px;background:var(--bg);color:var(--fg);font:15px/1.45 system-ui,Segoe UI,Roboto,Arial}
h1{margin:0 0 8px;font-weight:800}
h2{margin:10px 0 6px;font-size:16px}
.card{background:var(--muted);border-radius:14px;padding:14px;box-shadow:0 2px 10px rgba(0,0,0,.25)}
.row{display:flex;flex-wrap:wrap;gap:12px}
.col{flex:1;min-width:280px}
label{display:block;font-size:12px;opacity:.9;margin-bottom:6px}
input,textarea,select,button{width:100%;border:0;border-radius:10px;padding:10px 12px;background:#141427;color:var(--fg)}
textarea{min-height:110px;resize:vertical}
button{cursor:pointer;font-weight:700;background:linear-gradient(90deg,var(--accent),var(--gold))}
button.ghost{background:#22223a}
.small{font-size:12px;opacity:.8}
.pill{display:inline-block;padding:4px 10px;border-radius:999px;background:#22223a;font-size:12px}
.grid2{display:grid;grid-template-columns:1fr 1fr;gap:8px}
.list{list-style:none;padding:0;margin:6px 0 0}
.list li{background:#16162a;margin:6px 0;padding:8px 10px;border-radius:8px;display:flex;justify-content:space-between;align-items:center;gap:8px}
.badge{font-size:11px;background:#26263d;border-radius:8px;padding:3px 6px}
.pickbox{background:#131323;border-radius:10px;padding:10px;white-space:pre-wrap}
table{width:100%;border-collapse:collapse}
th,td{border-bottom:1px solid #28283f;padding:8px 6px;text-align:left}
</style>
</head>
<body>
<h1>⚔️ Jank One 🥔🍟 League Matchup Draft 🎲</h1>

<div class="row">
  <div class="col card">
    <h2>Setup</h2>
    <div class="grid2">
      <div>
        <label>Team A name</label>
        <input id="teamAName" placeholder="TeamHAT"/>
      </div>
      <div>
        <label>Team B name</label>
        <input id="teamBName" placeholder="Jank One"/>
      </div>
    </div>
    <div class="grid2" style="margin-top:8px">
      <div>
        <label>Slots per team (number of picks)</label>
        <input id="slots" type="number" min="1" value="4"/>
      </div>
      <div>
        <label>Allow duplicate Gods per team?</label>
        <select id="dupeGods">
          <option value="no">No (unique gods per team)</option>
          <option value="yes">Yes (duplicates allowed)</option>
        </select>
      </div>
    </div>
    <div style="margin-top:8px">
      <label>Available Gods (comma separated)</label>
      <input id="gods" value="War,Nature,Magic,Light,Death,Deception"/>
    </div>
    <div class="grid2" style="margin-top:8px">
      <div>
        <label>Team A roster (one per line)</label>
        <textarea id="rosterA" placeholder="davuss&#10;dainos&#10;grizzly"></textarea>
      </div>
      <div>
        <label>Team B roster (one per line)</label>
        <textarea id="rosterB" placeholder="0din&#10;Ghost&#10;Famo"></textarea>
      </div>
    </div>
    <div class="grid2" style="margin-top:8px">
      <button id="coin">🪙 Coin Flip for First Pick</button>
      <span id="coinResult" class="pill">no flip yet</span>
    </div>
    <div class="grid2" style="margin-top:8px">
      <button class="ghost" id="makeOrder">Generate Snake Order</button>
      <button class="ghost" id="reset">Reset</button>
    </div>
    <div style="margin-top:8px">
      <label>Pick Order</label>
      <div id="orderBox" class="pickbox"></div>
    </div>
  </div>

  <div class="col card">
    <h2>On the Clock</h2>
    <div class="grid2">
      <div>
        <label>Captain</label>
        <input id="currentCaptain" readonly>
      </div>
      <div>
        <label>Pick #</label>
        <input id="currentPick" readonly>
      </div>
    </div>
    <div class="grid2" style="margin-top:8px">
      <div>
        <label>Choose Player (from this team)</label>
        <select id="pickPlayer"></select>
      </div>
      <div>
        <label>Assign God</label>
        <select id="pickGod"></select>
      </div>
    </div>
    <div class="grid2" style="margin-top:8px">
      <button id="confirmPick">Confirm Pick</button>
      <button class="ghost" id="skipPick">Skip</button>
    </div>

    <div style="margin-top:10px">
      <label>Team A picks</label>
      <ul id="listA" class="list"></ul>
      <label>Team B picks</label>
      <ul id="listB" class="list"></ul>
    </div>
  </div>
</div>

<div class="row" style="margin-top:12px">
  <div class="col card">
    <h2>Matchups (A1 vs B1, A2 vs B2…)</h2>
    <table id="matchesTbl">
      <thead><tr><th>#</th><th id="thA">Team A</th><th id="thB">Team B</th></tr></thead>
      <tbody></tbody>
    </table>
    <div class="grid2" style="margin-top:8px">
      <button class="ghost" id="exportJson">Export JSON</button>
      <button class="ghost" id="exportCsv">Export CSV</button>
    </div>
    <p class="small">Tip: finish all picks to auto-generate the matchup table.</p>
  </div>
</div>

<script>
const $ = id => document.getElementById(id);

// ---- State ----
const S = {
  A:{name:'Team A', roster:[], picks:[], godsUsed:[]},
  B:{name:'Team B', roster:[], picks:[], godsUsed:[]},
  gods:[], allowDupe:false, slots:4,
  order:[], on:0, first:'A'
};

// ---- Helpers ----
const cleanList = txt => [...new Set(
  txt.split('\n').map(s=>s.trim()).filter(Boolean)
)];
const setTitles = ()=>{
  S.A.name = $('teamAName').value.trim()||'Team A';
  S.B.name = $('teamBName').value.trim()||'Team B';
  $('thA').textContent = S.A.name;
  $('thB').textContent = S.B.name;
};
const refreshGodSelect = (teamKey)=>{
  const team = S[teamKey];
  const used = S.allowDupe? [] : team.godsUsed;
  const avail = S.gods.filter(g=>!used.includes(g));
  $('pickGod').innerHTML = avail.map(g=>`<option>${g}</option>`).join('') || `<option disabled>No gods left</option>`;
};
const refreshPlayerSelect = (teamKey)=>{
  const pool = S[teamKey].roster.filter(p => !S[teamKey].picks.find(x=>x.player===p));
  $('pickPlayer').innerHTML = pool.map(p=>`<option>${p}</option>`).join('') || `<option disabled>No players left</option>`;
};
const renderOrder = ()=>{
  const names = {A:S.A.name,B:S.B.name};
  const out = S.order.map((k,i)=> (i===S.on?'👉 ':'  ')+ (i+1)+'. '+names[k]).join('\n');
  $('orderBox').textContent = out;
  $('currentCaptain').value = names[S.order[S.on]||''];
  $('currentPick').value = (S.on+1)+' / '+S.order.length;
};
const renderPicks = ()=>{
  const mk = (arr,el)=> {
    el.innerHTML = '';
    arr.forEach((x,i)=>{
      const li = document.createElement('li');
      li.innerHTML = `<span>${i+1}. ${x.player}</span><span class="badge">${x.god}</span>`;
      el.appendChild(li);
    });
  };
  mk(S.A.picks, $('listA'));
  mk(S.B.picks, $('listB'));
};
const buildMatches = ()=>{
  const n = Math.min(S.A.picks.length, S.B.picks.length);
  const tb = $('matchesTbl').querySelector('tbody');
  tb.innerHTML = '';
  for(let i=0;i<n;i++){
    const a = S.A.picks[i], b = S.B.picks[i];
    const tr = document.createElement('tr');
    tr.innerHTML = `<td>${i+1}</td><td>${a.player} <span class="badge">${a.god}</span></td><td>${b.player} <span class="badge">${b.god}</span></td>`;
    tb.appendChild(tr);
  }
};

// ---- Core actions ----
function loadSetup(){
  setTitles();
  S.slots = Math.max(1, parseInt($('slots').value||4,10));
  S.allowDupe = ($('dupeGods').value==='yes');
  S.gods = $('gods').value.split(',').map(s=>s.trim()).filter(Boolean);
  S.A.roster = cleanList($('rosterA').value);
  S.B.roster = cleanList($('rosterB').value);
  // reset picks if rosters changed
  S.A.picks = S.A.picks.filter(p=>S.A.roster.includes(p.player));
  S.B.picks = S.B.picks.filter(p=>S.B.roster.includes(p.player));
  S.A.godsUsed = S.A.picks.map(p=>p.god);
  S.B.godsUsed = S.B.picks.map(p=>p.god);
  renderPicks(); buildMatches();
}

function coinFlip(){
  loadSetup();
  S.first = Math.random()<0.5 ? 'A':'B';
  $('coinResult').textContent = (S.first==='A'?S.A.name:S.B.name)+' starts';
}

function makeOrder(){
  loadSetup();
  const rounds = S.slots; // picks per team
  let order=[];
  for(let i=0;i<rounds;i++){
    const seq = (i%2===0)? ['A','B'] : ['B','A']; // snake
    order = order.concat(seq);
  }
  // ensure first pick belongs to coin winner
  if(S.first==='B'){ order = order.map(v=>v==='A'?'B':'A'); }
  S.order = order; S.on = 0;
  renderOrder();
  // prepare selects for first captain
  const teamKey = S.order[S.on];
  refreshPlayerSelect(teamKey);
  refreshGodSelect(teamKey);
}

function nextTurn(){
  if(S.on < S.order.length-1){ S.on++; }
  renderOrder();
  const teamKey = S.order[S.on];
  if(!teamKey){ return; }
  refreshPlayerSelect(teamKey);
  refreshGodSelect(teamKey);
}

function confirmPick(){
  if(!S.order.length) return alert('Generate the snake order first.');
  const teamKey = S.order[S.on];
  const team = S[teamKey];
  const player = $('pickPlayer').value;
  const god = $('pickGod').value;
  if(!player || !god) return;
  // validations
  if(team.picks.find(x=>x.player===player)) return alert('Player already picked.');
  if(!S.allowDupe && team.godsUsed.includes(god)) return alert('God already used by this team.');
  team.picks.push({player, god});
  team.godsUsed.push(god);
  renderPicks(); buildMatches();
  nextTurn();
}

function skipPick(){ nextTurn(); }

function resetAll(){
  if(!confirm('Reset everything?')) return;
  Object.assign(S,{
    A:{name:'Team A', roster:[], picks:[], godsUsed:[]},
    B:{name:'Team B', roster:[], picks:[], godsUsed:[]},
    gods:[], allowDupe:false, slots:4, order:[], on:0, first:'A'
  });
  $('teamAName').value=''; $('teamBName').value='';
  $('rosterA').value=''; $('rosterB').value='';
  $('gods').value='War,Nature,Magic,Light,Death,Deception';
  $('slots').value='4'; $('dupeGods').value='no';
  $('coinResult').textContent='no flip yet';
  $('orderBox').textContent='';
  $('currentCaptain').value=''; $('currentPick').value='';
  $('listA').innerHTML=''; $('listB').innerHTML='';
  $('matchesTbl').querySelector('tbody').innerHTML='';
}

// ---- Export ----
function exportJSON(){
  const payload = {
    teamA:S.A.name, teamB:S.B.name,
    slots:S.slots, allowDuplicateGods:S.allowDupe,
    godsPool:S.gods,
    picksA:S.A.picks, picksB:S.B.picks,
    order:S.order, first:S.first,
    timestamp:new Date().toISOString()
  };
  const blob = new Blob([JSON.stringify(payload,null,2)],{type:'application/json'});
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'jank_draft_gods.json';
  a.click();
}
function exportCSV(){
  const n = Math.max(S.A.picks.length, S.B.picks.length);
  let rows = [['#','Team A Player','God A','Team B Player','God B']];
  for(let i=0;i<n;i++){
    const a = S.A.picks[i]||{player:'',god:''};
    const b = S.B.picks[i]||{player:'',god:''};
    rows.push([i+1, a.player, a.god, b.player, b.god]);
  }
  const csv = rows.map(r=>r.map(x=>`"${String(x).replace(/"/g,'""')}"`).join(',')).join('\n');
  const blob = new Blob([csv],{type:'text/csv'});
  const a = document.createElement('a');
  a.href = URL.createObjectURL(blob);
  a.download = 'matchups.csv';
  a.click();
}

// ---- Wire up ----
['teamAName','teamBName','rosterA','rosterB','slots','dupeGods','gods']
.forEach(id=>$(id).addEventListener('input', loadSetup));

$('coin').onclick = coinFlip;
$('makeOrder').onclick = makeOrder;
$('confirmPick').onclick = confirmPick;
$('skipPick').onclick = skipPick;
$('reset').onclick = resetAll;
$('exportJson').onclick = exportJSON;
$('exportCsv').onclick = exportCSV;

// Initial
loadSetup();
</script>
</body>
</html>
