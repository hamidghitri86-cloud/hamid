<!DOCTYPE html>
<html lang="ar">
<head>
<meta charset="UTF-8" />
<meta name="viewport" content="width=device-width, initial-scale=1" />
<title>Hamid - Production Tracker</title>
<style>
  body { font-family: Arial, sans-serif; padding:20px; background:#f6f9fc; color:#222; direction: ltr; }
  .container { max-width:1100px; margin:0 auto; }
  .card { background:#fff; border-radius:8px; padding:12px 16px; box-shadow:0 2px 6px rgba(20,30,50,0.08); margin-bottom:12px; }
  h1,h2{margin:0 0 8px 0; color:#173f3f;}
  label{display:inline-block; margin:6px 4px; font-size:14px;}
  input, select, button { padding:6px 8px; margin:4px; border-radius:4px; border:1px solid #cfd8dc; }
  table{width:100%; border-collapse:collapse; margin-top:10px;}
  th,td{border:1px solid #e0e6ea; padding:8px; text-align:center;}
  th{background:#2e7d32; color:#fff;}
  .highDefect{background:#ffd6d6;}
  .hidden{display:none;}
  .topbar{display:flex; justify-content:space-between; align-items:center;}
  .btn{background:#1976d2; color:#fff; border:none; cursor:pointer; padding:6px 12px; margin:2px; border-radius:4px;}
  .btn.secondary{background:#555;}
  /* important: remove fixed small canvas size, we will size canvases specifically */
  canvas { max-width:100%; height:auto; display:block; margin:8px auto; }
  /* target bigger size for our specific charts */
  #dailyChart, #weeklyChart { width:450px; height:450px; }
  .center { text-align:center; }
  .cover-center { display:flex; align-items:center; justify-content:center; height:100vh; flex-direction:column; }
</style>
</head>
<body>
<div class="container">

<!-- LOGIN -->
<div id="loginDiv" class="card">
  <div class="topbar">
    <div>
      <h1>Production Tracker</h1>
      <p>Enter password to open the app. Default password: <strong> </strong></p>
    </div>
    <div>
      <label>Password:</label>
      <input id="passwordInput" type="password" placeholder="Enter password">
      <button class="btn" onclick="login()">Login</button>
      <p id="loginMsg" style="color:#c62828; margin-top:6px;"></p>
    </div>
  </div>
</div>

<!-- MAIN APP -->
<div id="mainDiv" class="hidden">

  <!-- Components -->
  <div class="card">
    <h1>Welcome to INATEL</h1>
    <h2>Add / Select Component</h2>
    <input id="newComp" type="text" placeholder="Add new component">
    <button class="btn" onclick="addComponent()">Add Component</button>
    <br><label>Select component:</label>
    <select id="compSelect"></select>
  </div>

  <!-- Daily Entry -->
  <div class="card">
    <h2>Daily Entry</h2>
    <label>Date:</label><input id="entryDate" type="date">
    <label>Time:</label><input id="entryTime" type="time">
    <label>Qty In:</label><input id="qtyIn" type="number" min="0">
    <label>Qty Out:</label><input id="qtyOut" type="number" min="0">
    <label>Qty Defective:</label><input id="qtyDef" type="number" min="0">
    <label>Comment:</label><input id="defComment" type="text" placeholder="Why defective?">
    <br>
    <button class="btn" onclick="addEntry()">Add Entry</button>
  </div>

  <!-- Entries Table -->
  <div class="card">
    <h2>Entries (Edit/Delete)</h2>
    <table id="entryTable">
      <tr><th>Date</th><th>Time</th><th>Component</th><th>In</th><th>Out</th><th>Def</th><th>Comment</th><th>Actions</th></tr>
    </table>
  </div>

  <!-- Daily Stats -->
  <div class="card">
    <h2>Daily Stats</h2>
    <label>Date:</label><input id="statsDate" type="date">
    <button class="btn" onclick="showStats()">Show Stats</button>
    <button class="btn secondary" onclick="exportPDF('daily')">Save PDF</button>
    <table id="statsTable">
      <tr><th>Component</th><th>Qty In</th><th>Qty Out</th><th>Qty Def</th><th>Defect Rate (%)</th><th>Comments</th></tr>
    </table>
    <div class="center">
      <canvas id="dailyChart" width="450" height="450"></canvas>
    </div>
  </div>

  <!-- Weekly Stats -->
  <div class="card">
    <h2>Weekly Stats</h2>
    <label>Week start date:</label><input id="weekStart" type="date">
    <button class="btn" onclick="showWeeklyStats()">Show Weekly Stats</button>
    <button class="btn secondary" onclick="exportPDF('weekly')">Save PDF</button>
    <table id="weeklyTable">
      <tr><th>Component</th><th>Total In</th><th>Total Out</th><th>Total Def</th><th>Defect Rate (%)</th><th>Week Start</th><th>Week End</th></tr>
    </table>
    <div class="center">
      <canvas id="weeklyChart" width="450" height="450"></canvas>
    </div>
  </div>

</div>
</div>

<!-- Scripts -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/html2canvas/1.4.1/html2canvas.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/jspdf/2.5.1/jspdf.umd.min.js"></script>
<script>
/* --------- Config --------- */
const PASSWORD="3";
const DEFECT_THRESHOLD=5;

/* --------- Helpers --------- */
function todayISO(){ let d=new Date(); d.setMinutes(d.getMinutes()-d.getTimezoneOffset()); return d.toISOString().slice(0,10); }

/* --------- Login & Init --------- */
function login(){
  const pass=document.getElementById('passwordInput').value;
  if(pass===PASSWORD){
    document.getElementById('loginDiv').classList.add('hidden');
    document.getElementById('mainDiv').classList.remove('hidden');
    initApp();
  } else {
    document.getElementById('loginMsg').textContent="Wrong password!";
  }
}

/* --------- Data (localStorage) --------- */
let components=JSON.parse(localStorage.getItem('components'))||[];
let entries=JSON.parse(localStorage.getItem('entries'))||[];
function saveData(){ localStorage.setItem('components',JSON.stringify(components)); localStorage.setItem('entries',JSON.stringify(entries)); }

/* --------- Init --------- */
function initApp(){
  document.getElementById('entryDate').value=todayISO();
  document.getElementById('statsDate').value=todayISO();
  document.getElementById('weekStart').value=todayISO();
  document.getElementById('entryTime').value="08:00";
  refreshCompSelect();
  renderEntryTable();
  // initial empty charts
  renderPieChart('dailyChart',[],[]);
  renderPieChart('weeklyChart',[],[]);
}

/* --------- Components --------- */
function addComponent(){
  const comp=document.getElementById('newComp').value.trim();
  if(comp && !components.includes(comp)){
    components.push(comp);
    saveData();
    refreshCompSelect();
    document.getElementById('newComp').value='';
  } else {
    alert("Invalid or duplicate component");
  }
}
function refreshCompSelect(){
  const sel=document.getElementById('compSelect'); sel.innerHTML='';
  if(components.length===0){
    // Add a default example component to avoid empty select
    components.push('ExampleComp');
    saveData();
  }
  components.forEach(c=>{
    let o=document.createElement('option'); o.value=c; o.textContent=c; sel.appendChild(o);
  });
}

/* --------- Entries --------- */
function addEntry(){
  const comp=document.getElementById('compSelect').value;
  const date=document.getElementById('entryDate').value;
  const time=document.getElementById('entryTime').value;
  const qtyIn=parseInt(document.getElementById('qtyIn').value)||0;
  const qtyOut=parseInt(document.getElementById('qtyOut').value)||0;
  const qtyDef=parseInt(document.getElementById('qtyDef').value)||0;
  const comment=document.getElementById('defComment').value||'';
  if(!comp||!date){ alert("Fill component and date"); return; }
  entries.push({date,time,comp,qtyIn,qtyOut,qtyDef,comment});
  saveData(); renderEntryTable();
  document.getElementById('qtyIn').value=''; document.getElementById('qtyOut').value=''; document.getElementById('qtyDef').value=''; document.getElementById('defComment').value='';
}

function renderEntryTable(){
  const table=document.getElementById('entryTable');
  table.innerHTML=`<tr><th>Date</th><th>Time</th><th>Component</th><th>In</th><th>Out</th><th>Def</th><th>Comment</th><th>Actions</th></tr>`;
  entries.forEach((e,i)=>{
    const row=table.insertRow();
    row.insertCell(0).textContent=e.date;
    row.insertCell(1).textContent=e.time;
    row.insertCell(2).textContent=e.comp;
    row.insertCell(3).textContent=e.qtyIn;
    row.insertCell(4).textContent=e.qtyOut;
    row.insertCell(5).textContent=e.qtyDef;
    row.insertCell(6).textContent=e.comment;
    const acts=row.insertCell(7);
    const edit=document.createElement('button'); edit.textContent='Edit'; edit.className='btn'; edit.style.marginRight='6px';
    edit.onclick=()=>{ 
      document.getElementById('entryDate').value=e.date;
      document.getElementById('entryTime').value=e.time;
      document.getElementById('compSelect').value=e.comp;
      document.getElementById('qtyIn').value=e.qtyIn;
      document.getElementById('qtyOut').value=e.qtyOut;
      document.getElementById('qtyDef').value=e.qtyDef;
      document.getElementById('defComment').value=e.comment;
      entries.splice(i,1);
      saveData();
      renderEntryTable();
    };
    const del=document.createElement('button'); del.textContent='Delete'; del.className='btn secondary';
    del.onclick=()=>{ if(confirm('Delete this entry?')){ entries.splice(i,1); saveData(); renderEntryTable(); } };
    acts.appendChild(edit); acts.appendChild(del);
  });
}

/* ====== Stats ====== */
let chartInstances={};

function showStats(){
  const date=document.getElementById('statsDate').value;
  const filtered=entries.filter(e=>e.date===date);
  const compStats={};
  filtered.forEach(e=>{
    if(!compStats[e.comp]) compStats[e.comp]={qtyIn:0,qtyOut:0,qtyDef:0,comments:[]};
    compStats[e.comp].qtyIn+=e.qtyIn;
    compStats[e.comp].qtyOut+=e.qtyOut;
    compStats[e.comp].qtyDef+=e.qtyDef;
    if(e.comment) compStats[e.comp].comments.push(e.comment);
  });
  const table=document.getElementById('statsTable'); table.innerHTML=`<tr><th>Component</th><th>Qty In</th><th>Qty Out</th><th>Qty Def</th><th>Defect Rate (%)</th><th>Comments</th></tr>`;
  const labels=[]; const data=[];
  for(const c in compStats){
    const inCount = compStats[c].qtyIn || 0;
    const r = inCount>0 ? ((compStats[c].qtyDef/inCount)*100) : 0;
    const rStr = r.toFixed(2);
    const row=table.insertRow();
    row.insertCell(0).textContent=c;
    row.insertCell(1).textContent=compStats[c].qtyIn;
    row.insertCell(2).textContent=compStats[c].qtyOut;
    row.insertCell(3).textContent=compStats[c].qtyDef;
    row.insertCell(4).textContent=rStr;
    row.insertCell(5).textContent=compStats[c].comments.join("; ");
    if(compStats[c].qtyDef>DEFECT_THRESHOLD) row.classList.add('highDefect');
    labels.push(c); data.push(Number(rStr));
  }
  renderPieChart('dailyChart',labels,data);
}

function showWeeklyStats(){
  const startDate=new Date(document.getElementById('weekStart').value);
  const endDate=new Date(startDate); endDate.setDate(startDate.getDate()+6);
  const filtered=entries.filter(e=>{
    const d=new Date(e.date); return d>=startDate && d<=endDate;
  });
  const compStats={};
  filtered.forEach(e=>{
    if(!compStats[e.comp]) compStats[e.comp]={qtyIn:0,qtyOut:0,qtyDef:0};
    compStats[e.comp].qtyIn+=e.qtyIn;
    compStats[e.comp].qtyOut+=e.qtyOut;
    compStats[e.comp].qtyDef+=e.qtyDef;
  });
  const table=document.getElementById('weeklyTable'); table.innerHTML=`<tr><th>Component</th><th>Total In</th><th>Total Out</th><th>Total Def</th><th>Defect Rate (%)</th><th>Week Start</th><th>Week End</th></tr>`;
  const labels=[]; const data=[];
  for(const c in compStats){
    const inCount = compStats[c].qtyIn || 0;
    const r = inCount>0 ? ((compStats[c].qtyDef/inCount)*100) : 0;
    const rStr = r.toFixed(2);
    const row=table.insertRow();
    row.insertCell(0).textContent=c;
    row.insertCell(1).textContent=compStats[c].qtyIn;
    row.insertCell(2).textContent=compStats[c].qtyOut;
    row.insertCell(3).textContent=compStats[c].qtyDef;
    row.insertCell(4).textContent=rStr;
    row.insertCell(5).textContent=startDate.toISOString().slice(0,10);
    row.insertCell(6).textContent=endDate.toISOString().slice(0,10);
    labels.push(c); data.push(Number(rStr));
  }
  renderPieChart('weeklyChart',labels,data);
}

/* ====== Charts (Chart.js) ====== */
function renderPieChart(canvasId, labels, data){
  // destroy old
  if(chartInstances[canvasId]) chartInstances[canvasId].destroy();
  const ctx = document.getElementById(canvasId).getContext('2d');
  // ensure canvas element has width/height attributes for high-res export
  const canvasEl = document.getElementById(canvasId);
  canvasEl.width = 450;
  canvasEl.height = 450;
  chartInstances[canvasId] = new Chart(ctx, {
    type: 'pie',
    data: {
      labels: labels,
      datasets: [{
        data: data,
        backgroundColor: labels.map((_,i)=>['#1976d2','#f44336','#ff9800','#4caf50','#9c27b0','#2196f3'][i%6])
      }]
    },
    options: {
      responsive: false,
      maintainAspectRatio: false,
      plugins: {
        title: {
          display: true,
          text: canvasId.includes('daily') ? 'Daily Defect Rate (%)' : 'Weekly Defect Rate (%)'
        },
        legend: { display: true, position: 'bottom' }
      }
    }
  });
}

/* ====== Export PDF (cover + table + chart) ====== */
async function exportPDF(type){
  // type can be 'daily' or 'weekly'
  const { jsPDF } = window.jspdf;
  const pdf = new jsPDF({ unit:'pt', format:'a4' }); // pt units
  const pageWidth = pdf.internal.pageSize.getWidth();
  const pageHeight = pdf.internal.pageSize.getHeight();

  // --- 1) Cover page ---
  // White background by default. Add INATEL in dark blue centered.
  const blueDark = '#0d47a1'; // dark blue professional
  pdf.setFillColor(255,255,255);
  pdf.rect(0,0,pageWidth,pageHeight,'F');
  pdf.setFontSize(48);
  pdf.setTextColor(13,71,161); // dark blue
  pdf.setFont("helvetica","bold");
  const title = "INATEL";
  const textWidth = pdf.getTextWidth(title);
  pdf.text(title, (pageWidth - textWidth) / 2, pageHeight / 2);
  // small gap, then date
  pdf.setFontSize(12);
  pdf.setTextColor(80,80,80);
  const today = new Date();
  const dateStr = today.toISOString().slice(0,10);
  const dateText = "Report date: " + dateStr;
  const dateWidth = pdf.getTextWidth(dateText);
  pdf.text(dateText, (pageWidth - dateWidth) / 2, pageHeight / 2 + 24);

  // --- Prepare data for table/chart capture ---
  // Make sure charts are rendered/up-to-date:
  if(type==='daily'){
    showStats();
  } else if(type==='weekly'){
    showWeeklyStats();
  }

  // Wait a tick so charts update visually
  await new Promise(r=>setTimeout(r,300));

  // --- 2) Table page ---
  pdf.addPage();
  // capture the relevant table element to image using html2canvas
  const tableEl = (type==='daily') ? document.getElementById('statsTable') : document.getElementById('weeklyTable');
  // Improve table style for screenshot: clone and scale to a larger temporary element
  const clonedTable = tableEl.cloneNode(true);
  // If table empty (only header), add a "No data" row
  if(clonedTable.rows.length <= 1){
    const nr = clonedTable.insertRow();
    const nc = nr.insertCell(0);
    nc.colSpan = clonedTable.rows[0].cells.length;
    nc.textContent = 'No data for selected period';
    nc.style.textAlign = 'center';
  }
  // Prepare wrapper
  const wrapper = document.createElement('div');
  wrapper.style.width = '800px';
  wrapper.style.padding = '20px';
  wrapper.style.background = '#fff';
  const heading = document.createElement('h2');
  heading.textContent = (type==='daily') ? 'Daily Stats' : 'Weekly Stats';
  heading.style.color = '#173f3f';
  heading.style.fontFamily = 'Arial, sans-serif';
  wrapper.appendChild(heading);
  wrapper.appendChild(clonedTable);
  document.body.appendChild(wrapper); // append temporarily for html2canvas

  // use html2canvas to rasterize wrapper
  const canvasTable = await html2canvas(wrapper, { scale: 2 });
  document.body.removeChild(wrapper);

  const imgTableData = canvasTable.toDataURL('image/png');
  // Fit the table image into pdf with margin
  const margin = 40;
  const maxWidth = pageWidth - margin*2;
  const imgProps = pdf.getImageProperties(imgTableData);
  const imgHeight = (imgProps.height * maxWidth) / imgProps.width;
  pdf.addImage(imgTableData, 'PNG', margin, 40, maxWidth, imgHeight);

  // --- 3) Chart page ---
  pdf.addPage();
  // get chart canvas
  const chartCanvas = (type==='daily') ? document.getElementById('dailyChart') : document.getElementById('weeklyChart');

  // For Chart.js canvas: get dataURL directly
  let chartDataURL = null;
  // If Chart instance exists, use its canvas.toDataURL (higher resolution if possible)
  try{
    // temporarily increase pixel ratio to capture higher quality image
    chartDataURL = chartCanvas.toDataURL('image/png');
  } catch(err){
    // fallback: html2canvas
    const tmp = await html2canvas(chartCanvas, { scale: 2 });
    chartDataURL = tmp.toDataURL('image/png');
  }

  // Add small heading
  pdf.setFontSize(18);
  pdf.setTextColor(23,63,63);
  pdf.text((type==='daily') ? 'Daily Defect Rate' : 'Weekly Defect Rate', margin, 50);

  // Add the chart image centered
  const chartMaxWidth = pageWidth - margin*2;
  // Create an Image object to get dimensions
  const img = new Image();
  await new Promise((resolve) => {
    img.onload = () => { resolve(); };
    img.onerror = ()=>{ resolve(); };
    img.src = chartDataURL;
  });
  // compute height to keep aspect ratio
  const chartImgPropsWidth = img.width || 450;
  const chartImgPropsHeight = img.height || 450;
  const chartRenderHeight = (chartImgPropsHeight * chartMaxWidth) / chartImgPropsWidth;
  pdf.addImage(chartDataURL, 'PNG', margin, 80, chartMaxWidth, chartRenderHeight);

  // --- Footer on last page ---
  pdf.setFontSize(10);
  pdf.setTextColor(120,120,120);
  pdf.text("Prepared by: INATEL", margin, pageHeight - 30);
  pdf.text("Generated: " + dateStr, pageWidth - margin - pdf.getTextWidth("Generated: " + dateStr), pageHeight - 30);

  // Save file
  pdf.save(`${type}_report_${dateStr}.pdf`);
}

/* ====== Print (optional) ====== */
function printStats(type){
  const table = type==='daily'? document.getElementById('statsTable') : document.getElementById('weeklyTable');
  const w=window.open('','Print','width=900,height=600');
  w.document.write('<html><head><title>Print</title></head><body>');
  w.document.write(table.outerHTML);
  w.document.write('</body></html>');
  w.document.close();
  w.print();
}
</script>
</body>
</html>
