# hamid
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Hamid </title>
<style>
  body { font-family: Arial, padding: 20px; background:#f6f9fc; color:#222; }
  .container { max-width:1100px; margin:0 auto; }
  .card { background:#fff; border-radius:8px; padding:12px 16px; box-shadow:0 2px 6px rgba(20,30,50,0.08); margin-bottom:12px; }
  h1{margin:0 0 8px 0; color:#173f3f;}
  label{display:inline-block; margin:6px 4px; font-size:14px;}
  input, select, button { padding:6px 8px; margin:4px; border-radius:4px; border:1px solid #cfd8dc; }
  table{width:100%; border-collapse:collapse; margin-top:10px;}
  th,td{border:1px solid #e0e6ea; padding:8px; text-align:center;}
  th{background:#2e7d32; color:#fff;}
  .highDefect{background:#ffd6d6;}
  .hidden{display:none;}
  .notice{color:#055; font-size:14px; margin-bottom:8px;}
  .topbar{display:flex; justify-content:space-between; align-items:center;}
  .btn{background:#1976d2; color:#fff; border:none;}
  .btn.secondary{background:#555;}
</style>
</head>
<body>
<div class="container">

  <div id="loginDiv" class="card">
    <div class="topbar">
      <div>
        <h1>Hamid Production Tracker — Login</h1>
        <p class="notice">Enter the password to open the app<strong></strong></p>
      </div>
      <div>
        <label>Password:</label>
        <input id="passwordInput" type="password" placeholder="Enter password">
        <button class="btn" onclick="login()">Login</button>
        <p id="loginMsg" style="color:#c62828; margin-top:6px;"></p>
      </div>
    </div>
  </div>

  <div id="mainDiv" class="hidden">
    <div class="card">
      <h1>Hamid Production Tracker</h1>
      <p class="notice">Site runs fully offline using browser storage. To change password open file and edit <code>const PASSWORD</code>.</p>
    </div>

    <div class="card">
      <h2>Add / Select Component</h2>
      <input id="newComp" type="text" placeholder="Add new component">
      <button class="btn" onclick="addComponent()">Add Component</button>
      <br><label>Select component:</label>
      <select id="compSelect"></select>
    </div>

    <div class="card">
      <h2>Daily Entry</h2>
      <label>Date:</label><input id="entryDate" type="date">
      <label>Qty In:</label><input id="qtyIn" type="number" min="0">
      <label>Qty Out:</label><input id="qtyOut" type="number" min="0">
      <label>Qty Defective:</label><input id="qtyDef" type="number" min="0">
      <label>Comment:</label><input id="defComment" type="text" placeholder="Why defective?">
      <br>
      <button class="btn" onclick="addEntry()">Add Entry</button>
      
    </div>

    <div class="card">
      <h2>Entries (Edit/Delete)</h2>
      <table id="entryTable">
        <tr><th>Date</th><th>Component</th><th>In</th><th>Out</th><th>Def</th><th>Comment</th><th>Actions</th></tr>
      </table>
    </div>

    <div class="card">
      <h2>Daily Stats</h2>
      <label>Date:</label><input id="statsDate" type="date">
      <button class="btn" onclick="showStats()">Show Stats</button>
      <table id="statsTable">
        <tr><th>Component</th><th>Qty In</th><th>Qty Out</th><th>Qty Def</th><th>Defect Rate (%)</th><th>Comments</th></tr>
      </table>
      <p id="totalStats"></p>
    </div>

    <div class="card">
      <h2>Weekly Stats</h2>
      <label>Week start date:</label><input id="weekStart" type="date">
      <button class="btn" onclick="showWeeklyStats()">Show Weekly Stats</button>
      <table id="weeklyTable">
        <tr><th>Component</th><th>Total In</th><th>Total Out</th><th>Total Def</th><th>Defect Rate (%)</th><th>Week Start</th><th>Week End</th></tr>
      </table>
    </div>
  </div>

</div>

<script>
/* ====== Config ====== */
const PASSWORD = "0782695244"; // change here if you want a different password
const DEFECT_THRESHOLD = 5; // percent

/* ====== Helpers ====== */
function todayISO(){ let d=new Date(); d.setMinutes(d.getMinutes()-d.getTimezoneOffset()); return d.toISOString().slice(0,10); }

/* ====== Login ====== */
function login(){
  const pass = document.getElementById('passwordInput').value;
  if(pass === PASSWORD){ 
    document.getElementById('loginDiv').classList.add('hidden');
    document.getElementById('mainDiv').classList.remove('hidden');
    initApp();
  } else {
    document.getElementById('loginMsg').textContent = "Wrong password!";
  }
}

/* ====== Data ====== */
let components = JSON.parse(localStorage.getItem('components')) || [];
let entries = JSON.parse(localStorage.getItem('entries')) || [];

function saveData(){
  localStorage.setItem('components', JSON.stringify(components));
  localStorage.setItem('entries', JSON.stringify(entries));
}

/* ====== Init ====== */
function initApp(){
  // put today's date in fields to help user
  document.getElementById('entryDate').value = todayISO();
  document.getElementById('statsDate').value = todayISO();
  document.getElementById('weekStart').value = todayISO();
  refreshCompSelect();
  renderEntryTable();
}

/* ====== Components ====== */
function addComponent(){
  const comp = document.getElementById('newComp').value.trim();
  if(comp && !components.includes(comp)){ components.push(comp); saveData(); refreshCompSelect(); document.getElementById('newComp').value=''; }
  else alert("Invalid or duplicate component");
}
function refreshCompSelect(){
  const sel = document.getElementById('compSelect');
  sel.innerHTML = '';
  components.forEach(c => { let o=document.createElement('option'); o.value=c; o.textContent=c; sel.appendChild(o); });
}

/* ====== Entries ====== */
function addEntry(){
  const comp = document.getElementById('compSelect').value;
  const date = document.getElementById('entryDate').value;
  const qtyIn = parseInt(document.getElementById('qtyIn').value) || 0;
  const qtyOut = parseInt(document.getElementById('qtyOut').value) || 0;
  const qtyDef = parseInt(document.getElementById('qtyDef').value) || 0;
  const comment = document.getElementById('defComment').value || '';
  if(!comp || !date){ alert("Fill component and date"); return; }
  entries.push({date, comp, qtyIn, qtyOut, qtyDef, comment});
  saveData(); renderEntryTable();
  // clear inputs
  document.getElementById('qtyIn').value=''; document.getElementById('qtyOut').value=''; document.getElementById('qtyDef').value=''; document.getElementById('defComment').value='';
}
function renderEntryTable(){
  const table = document.getElementById('entryTable');
  table.innerHTML = `<tr><th>Date</th><th>Component</th><th>In</th><th>Out</th><th>Def</th><th>Comment</th><th>Actions</th></tr>`;
  entries.forEach((e,i)=>{
    const row = table.insertRow();
    row.insertCell(0).textContent=e.date;
    row.insertCell(1).textContent=e.comp;
    row.insertCell(2).textContent=e.qtyIn;
    row.insertCell(3).textContent=e.qtyOut;
    row.insertCell(4).textContent=e.qtyDef;
    row.insertCell(5).textContent=e.comment;
    const acts = row.insertCell(6);
    const del = document.createElement('button'); del.textContent='Delete'; del.onclick = ()=>{ if(confirm('Delete this entry?')){ entries.splice(i,1); saveData(); renderEntryTable(); } };
    const edit = document.createElement('button'); edit.textContent='Edit'; edit.onclick = ()=>{ // load into form and remove old entry
      document.getElementById('entryDate').value=e.date; document.getElementById('compSelect').value=e.comp;
      document.getElementById('qtyIn').value=e.qtyIn; document.getElementById('qtyOut').value=e.qtyOut; document.getElementById('qtyDef').value=e.qtyDef; document.getElementById('defComment').value=e.comment;
      entries.splice(i,1); saveData(); renderEntryTable();
    };
    acts.appendChild(edit); acts.appendChild(del);
  });
}

/* ====== Daily Stats (group by comp and sum) ====== */
function showStats(){
  const date = document.getElementById('statsDate').value;
  const table = document.getElementById('statsTable');
  table.innerHTML = `<tr><th>Component</th><th>Qty In</th><th>Qty Out</th><th>Qty Def</th><th>Defect Rate (%)</th><th>Comments</th></tr>`;
  const daily = entries.filter(e => e.date === date);
  const grouped = {};
  daily.forEach(e => {
    if(!grouped[e.comp]) grouped[e.comp]={qtyIn:0,qtyOut:0,qtyDef:0,comments:[]};
    grouped[e.comp].qtyIn += e.qtyIn; grouped[e.comp].qtyOut += e.qtyOut; grouped[e.comp].qtyDef += e.qtyDef;
    if(e.comment) grouped[e.comp].comments.push(e.comment);
  });
  let totalIn=0,totalDef=0;
  for(const comp in grouped){
    const g = grouped[comp];
    const rate = g.qtyIn ? ((g.qtyDef/g.qtyIn)*100).toFixed(2) : '0.00';
    const row = table.insertRow();
    row.insertCell(0).textContent=comp; row.insertCell(1).textContent=g.qtyIn; row.insertCell(2).textContent=g.qtyOut;
    row.insertCell(3).textContent=g.qtyDef; row.insertCell(4).textContent=rate; row.insertCell(5).textContent=g.comments.join('; ');
    if(parseFloat(rate) > DEFECT_THRESHOLD) row.classList.add('highDefect');
    totalIn += g.qtyIn; totalDef += g.qtyDef;
  }
  const overall = totalIn ? ((totalDef/totalIn)*100).toFixed(2) : '0.00';
  document.getElementById('totalStats').textContent = `Total In: ${totalIn} | Total Defective: ${totalDef} | Overall Defect Rate: ${overall}%`;
}

/* ====== Weekly Stats (start -> start+6 days) ====== */
function showWeeklyStats(){
  const startVal = document.getElementById('weekStart').value;
  if(!startVal){ alert('Choose a week start date'); return; }
  const start = new Date(startVal); const end = new Date(start); end.setDate(end.getDate()+6);
  const weekly = entries.filter(e => { const d = new Date(e.date); return d >= start && d <= end; });
  const grouped = {};
  weekly.forEach(e => {
    if(!grouped[e.comp]) grouped[e.comp]={qtyIn:0,qtyOut:0,qtyDef:0};
    grouped[e.comp].qtyIn += e.qtyIn; grouped[e.comp].qtyOut += e.qtyOut; grouped[e.comp].qtyDef += e.qtyDef;
  });
  const table = document.getElementById('weeklyTable');
  table.innerHTML = `<tr><th>Component</th><th>Total In</th><th>Total Out</th><th>Total Def</th><th>Defect Rate (%)</th><th>Week Start</th><th>Week End</th></tr>`;
  for(const comp in grouped){
    const g = grouped[comp];
    const rate = g.qtyIn ? ((g.qtyDef/g.qtyIn)*100).toFixed(2) : '0.00';
    const row = table.insertRow();
    row.insertCell(0).textContent=comp; row.insertCell(1).textContent=g.qtyIn; row.insertCell(2).textContent=g.qtyOut;
    row.insertCell(3).textContent=g.qtyDef; row.insertCell(4).textContent=rate; row.insertCell(5).textContent=start.toLocaleDateString();
    row.insertCell(6).textContent=end.toLocaleDateString();
    if(parseFloat(rate) > DEFECT_THRESHOLD) row.classList.add('highDefect');
  }
}

/* ====== Utilities ====== */
function clearAll(){
  if(confirm('Clear ALL data (components + entries)? This cannot be undone.')){ components=[]; entries=[]; saveData(); refreshCompSelect(); renderEntryTable(); document.getElementById('statsTable').innerHTML=''; document.getElementById('weeklyTable').innerHTML=''; }
}

/* ====== Auto init if user saved login state (not implemented) ====== */
// nothing until login pressed

// For debugging: print to console so user knows file loaded
console.log('Hamid Tracker script loaded. Save file as hamid_tracker.html and open in browser. Default password = 1234.');
</script>
</body>
</html>
