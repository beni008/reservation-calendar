<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Apartment Reservation Calendar</title>
<!-- ical.js library to parse raw ICS feeds from Airbnb & Booking -->
<script src="https://cdnjs.cloudflare.com/ajax/libs/ical.js/1.5.0/ical.min.js"></script>
<style>
:root{--bg:#f4f6fb;--card:#fff;--primary:#2563eb;--danger:#dc2626;--success:#16a34a;--airbnb:#ff5a5f;--booking:#003580;--text:#172033;--muted:#64748b;--border:#dbe2ea}
*{box-sizing:border-box} body{margin:0;font-family:Arial,sans-serif;background:var(--bg);color:var(--text)}
header{background:#172033;color:#fff;padding:18px 24px} header h1{margin:0;font-size:22px} .wrap{max-width:1250px;margin:auto;padding:22px}
.grid{display:grid;grid-template-columns:330px 1fr;gap:20px}.card{background:var(--card);border-radius:14px;padding:18px;box-shadow:0 4px 18px #0000000d}
label{font-size:12px;font-weight:bold;display:block;margin:12px 0 5px;color:var(--muted)} input,select,button{width:100%;padding:10px;border:1px solid var(--border);border-radius:8px;font-size:14px}
button{background:var(--primary);color:#fff;border:none;cursor:pointer;font-weight:bold;margin-top:12px}button.secondary{background:#475569}button.danger{background:var(--danger)}button.success{background:var(--success)}
.topbar{display:flex;justify-content:space-between;align-items:center;margin-bottom:16px;gap:10px}.topbar select{max-width:280px}
.monthnav{display:flex;justify-content:space-between;align-items:center;margin-bottom:12px}.monthnav button{width:auto;margin:0;padding:9px 15px}
.weekdays,.calendar{display:grid;grid-template-columns:repeat(7,1fr);gap:5px}.weekdays div{text-align:center;font-size:12px;color:var(--muted);padding:6px}
.day{min-height:105px;background:#fff;border:1px solid var(--border);border-radius:9px;padding:7px;cursor:pointer;transition:.15s}.day:hover{border-color:var(--primary);transform:translateY(-1px)}
.day.empty{background:transparent;border:none;cursor:default}.day.overlap{background:#fee2e2;border:2px solid var(--danger)}.num{font-weight:bold;font-size:13px}
.res{margin-top:5px;border-radius:5px;padding:4px 5px;font-size:11px;white-space:nowrap;overflow:hidden;text-overflow:ellipsis;background:#dbeafe;color:#1e3a8a}
.res.manual{background:#dcfce7;color:#166534}
.res.airbnb{background:#ffe4e6;color:#9f1239}
.res.booking{background:#dbeafe;color:#1e40af}
.res.overlap{background:#fecaca;color:#991b1b;font-weight:bold}
.row{display:grid;grid-template-columns:1fr 1fr;gap:8px}.modal{display:none;position:fixed;inset:0;background:#0008;align-items:center;justify-content:center;padding:20px;z-index:10}.modal.show{display:flex}.modalbox{width:min(520px,100%);background:#fff;border-radius:14px;padding:20px;max-height:90vh;overflow:auto}.close{float:right;background:#e2e8f0;color:#111;width:auto;margin:0}.listitem{padding:9px;border-bottom:1px solid var(--border);font-size:13px}.small{font-size:12px;color:var(--muted);line-height:1.45}.sourcebox{background:#f8fafc;border-radius:8px;padding:10px;margin-top:10px}
@media(max-width:800px){.grid{grid-template-columns:1fr}.day{min-height:78px}.topbar{flex-direction:column;align-items:stretch}.topbar select{max-width:none}}
</style>
</head>
<body>
<header><h1>Apartment Reservation Calendar</h1></header>
<div class="wrap">
<div class="grid">
<aside class="card">
<h3>Listings</h3>
<select id="listingSelect"></select>
<label>New listing name</label><input id="newListing" placeholder="e.g. K01 Apartment">
<button onclick="addListing()">Add Listing</button>

<hr style="border:none;border-top:1px solid #e5e7eb;margin:20px 0">
<h3>Manual Reservation</h3>
<label>Guest name</label><input id="guest" placeholder="Guest name">
<div class="row"><div><label>Check-in</label><input id="checkin" type="date"></div><div><label>Check-out</label><input id="checkout" type="date"></div></div>
<button onclick="openDatePicker('checkin')">📅 Pick Check-in from Calendar</button>
<button class="secondary" onclick="openDatePicker('checkout')">📅 Pick Check-out from Calendar</button>
<button onclick="addReservation()">Add Manual Reservation</button>
<button class="success" onclick="exportManualICal()">📥 Export Manual iCal (.ics)</button>

<hr style="border:none;border-top:1px solid #e5e7eb;margin:20px 0">
<h3>Live Calendar Sources</h3>
<label>Airbnb iCal URL</label><input id="airbnbUrl" placeholder="https://www.airbnb.com/calendar/ical/...">
<label>Booking iCal URL</label><input id="bookingUrl" placeholder="https://admin.booking.com/hotel/...">
<button onclick="syncExternalFeeds()">🔄 Sync External Feeds</button>
<div class="sourcebox small">Uses standard client-side CORS proxying (`corsproxy.io`) to bypass browser cross-origin restrictions when importing directly from Airbnb & Booking.com.</div>
</aside>

<main class="card">
<div class="topbar"><div><b>Current listing:</b> <span id="listingName"></span></div><select id="quickListing"></select></div>
<div class="monthnav"><button onclick="changeMonth(-1)">← Previous</button><h2 id="monthTitle" style="margin:0;font-size:20px"></h2><button onclick="changeMonth(1)">Next →</button></div>
<div class="weekdays"><div>Mon</div><div>Tue</div><div>Wed</div><div>Thu</div><div>Fri</div><div>Sat</div><div>Sun</div></div>
<div class="calendar" id="calendar"></div>
<p class="small">Click any date to view details. Red squares indicate overlapping reservations across all sources.</p>
</main>
</div>
</div>

<div class="modal" id="modal"><div class="modalbox"><button class="close" onclick="closeModal()">Close</button><h3 id="modalTitle"></h3><div id="modalContent"></div></div></div>

<script>
const KEY='apartment_calendar_v1';
let data=JSON.parse(localStorage.getItem(KEY)||'null')||{listings:[{id:'default',name:"Kida's Apartments"}],reservations:[],sources:{}};
let selected=data.listings[0].id, view=new Date(); view.setDate(1);

function save(){localStorage.setItem(KEY,JSON.stringify(data))}
function esc(s){return String(s||'').replace(/[&<>"']/g,m=>({'&':'&amp;','<':'&lt;','>':'&gt;','"':'&quot;',"'":'&#39;'}[m]))}

function refreshListings(){
 const a=document.getElementById('listingSelect'),b=document.getElementById('quickListing');
 [a,b].forEach(x=>x.innerHTML='');
 data.listings.forEach(l=>[a,b].forEach(x=>x.add(new Option(l.name,l.id))));
 a.value=selected;b.value=selected;document.getElementById('listingName').textContent=(data.listings.find(x=>x.id===selected)||{}).name||'';
 loadSources();
}

document.getElementById('listingSelect').onchange=e=>{selected=e.target.value;refreshListings();render()};
document.getElementById('quickListing').onchange=e=>{selected=e.target.value;refreshListings();render()};

function addListing(){
 let n=document.getElementById('newListing').value.trim();if(!n)return alert('Enter a listing name');
 let id='l'+Date.now();data.listings.push({id,name:n});selected=id;document.getElementById('newListing').value='';
 save();refreshListings();render();
}

function dayKey(d){
 let yr=d.getFullYear(), mo=String(d.getMonth()+1).padStart(2,'0'), da=String(d.getDate()).padStart(2,'0');
 return `${yr}-${mo}-${da}`;
}
function parseDate(s){return new Date(s+'T00:00:00')}
function nights(r){let out=[];let d=parseDate(r.checkin),end=parseDate(r.checkout);while(d<end){out.push(dayKey(d));d.setDate(d.getDate()+1)}return out}
function activeRes(){return data.reservations.filter(r=>r.listing===selected)}

function addReservation(){
 let guest=document.getElementById('guest').value.trim()||'Manual reservation',ci=document.getElementById('checkin').value,co=document.getElementById('checkout').value;
 if(!ci||!co||parseDate(co)<=parseDate(ci))return alert('Check-out must be after check-in.');
 data.reservations.push({id:'r'+Date.now(),listing:selected,guest,checkin:ci,checkout:co,source:'manual'});
 save();document.getElementById('guest').value='';document.getElementById('checkin').value='';document.getElementById('checkout').value='';render();
}

function render(){
 refreshListings();let y=view.getFullYear(),m=view.getMonth();document.getElementById('monthTitle').textContent=view.toLocaleString('default',{month:'long',year:'numeric'});
 let cal=document.getElementById('calendar');cal.innerHTML='';
 let first=new Date(y,m,1),offset=(first.getDay()+6)%7;for(let i=0;i<offset;i++){let e=document.createElement('div');e.className='day empty';cal.appendChild(e)}
 let last=new Date(y,m+1,0).getDate(), rs=activeRes();
 for(let n=1;n<=last;n++){
  let d=new Date(y,m,n),key=dayKey(d),hits=rs.filter(r=>nights(r).includes(key));
  let box=document.createElement('div');box.className='day'+(hits.length>1?' overlap':'');
  box.innerHTML='<div class="num">'+n+'</div>'+hits.map(r=>'<div class="res '+(hits.length>1?'overlap':r.source)+'">'+esc(r.guest)+'</div>').join('');
  box.onclick=()=>showDate(key,hits);cal.appendChild(box);
 }
}

function changeMonth(x){view.setMonth(view.getMonth()+x);render()}

function showDate(key,hits){
 if(pickField){document.getElementById(pickField).value=key;pickField=null;return}
 document.getElementById('modalTitle').textContent='Reservations — '+key;
 let c=document.getElementById('modalContent');
 if(!hits.length)c.innerHTML='<p class="small">No reservation on this date.</p>';
 else c.innerHTML=hits.map(r=>'<div class="listitem"><b>'+esc(r.guest)+'</b><br>'+r.checkin+' → '+r.checkout+'<br><span class="small">Source: '+r.source+'</span>'+(r.source==='manual'?'<button class="danger" style="width:auto;padding:5px 8px;margin-left:8px" onclick="deleteReservation(\''+r.id+'\')">Delete</button>':'')+'</div>').join('');
 openModal();
}

function deleteReservation(id){if(confirm('Delete reservation?')){data.reservations=data.reservations.filter(r=>r.id!==id);save();closeModal();render()}}
function openModal(){document.getElementById('modal').classList.add('show')}
function closeModal(){document.getElementById('modal').classList.remove('show')}

let pickField=null;
function openDatePicker(field){
 document.getElementById('modalTitle').textContent='Select '+(field==='checkin'?'Check-in':'Check-out')+' date';
 document.getElementById('modalContent').innerHTML='<p class="small">Click a date in the main calendar after closing this window. Or use the date field directly.</p><button onclick="enablePick(\''+field+'\')">Select from calendar</button>';
 openModal();
}
function enablePick(f){pickField=f;closeModal();alert('Now click a date on the calendar.')}

function loadSources(){
 let s=(data.sources||{})[selected]||{};
 document.getElementById('airbnbUrl').value=s.airbnb||'';
 document.getElementById('bookingUrl').value=s.booking||'';
}

/* --- CORS PROXY ICAL SYNC LOGIC --- */
async function syncExternalFeeds(){
 let airbnbUrl = document.getElementById('airbnbUrl').value.trim();
 let bookingUrl = document.getElementById('bookingUrl').value.trim();

 if(!data.sources) data.sources = {};
 data.sources[selected] = { airbnb: airbnbUrl, booking: bookingUrl };
 save();

 // Remove existing fetched feeds (keep manual entries intact)
 data.reservations = data.reservations.filter(r => !(r.listing === selected && r.source !== 'manual'));

 const proxy = "https://corsproxy.io/?";

 if(airbnbUrl){
  await fetchAndParseICal(proxy + encodeURIComponent(airbnbUrl), 'airbnb', 'Airbnb Guest');
 }
 if(bookingUrl){
  await fetchAndParseICal(proxy + encodeURIComponent(bookingUrl), 'booking', 'Booking.com Guest');
 }

 save();
 render();
 alert('Calendar feeds synchronized successfully!');
}

async function fetchAndParseICal(proxyUrl, sourceKey, defaultTitle){
 try {
  let res = await fetch(proxyUrl);
  let text = await res.text();
  let jcalData = ICAL.parse(text);
  let comp = new ICAL.Component(jcalData);
  let vevents = comp.getAllSubcomponents('vevent');

  vevents.forEach(vevent => {
   let event = new ICAL.Event(vevent);
   let startDate = event.startDate.toJSDate();
   let endDate = event.endDate.toJSDate();

   data.reservations.push({
    id: 'ical_' + Date.now() + Math.random(),
    listing: selected,
    guest: event.summary || defaultTitle,
    checkin: dayKey(startDate),
    checkout: dayKey(endDate),
    source: sourceKey
   });
  });
 } catch(err) {
  console.error(`Error fetching/parsing ${sourceKey} feed:`, err);
  alert(`Failed to fetch ${sourceKey} calendar feed. Make sure the URL is valid.`);
 }
}

function exportManualICal(){
 let manualRes = activeRes().filter(r => r.source === 'manual');
 if(!manualRes.length) return alert('No manual reservations found for this listing.');

 let fmt = d => d.replace(/-/g, '');
 let now = new Date().toISOString().replace(/[-:]/g, '').split('.')[0] + 'Z';
 let activeName = (data.listings.find(x=>x.id===selected)||{}).name||'Listing';

 let lines = [
  'BEGIN:VCALENDAR',
  'VERSION:2.0',
  'PRODID:-//Apartment Reservation Calendar//EN',
  'CALSCALE:GREGORIAN',
  'METHOD:PUBLISH'
 ];

 manualRes.forEach(r => {
  lines.push(
   'BEGIN:VEVENT',
   `UID:${r.id}@reservationcalendar`,
   `DTSTAMP:${now}`,
   `DTSTART;VALUE=DATE:${fmt(r.checkin)}`,
   `DTEND;VALUE=DATE:${fmt(r.checkout)}`,
   `SUMMARY:Reserved - ${r.guest}`,
   'DESCRIPTION:Manual reservation',
   'STATUS:CONFIRMED',
   'END:VEVENT'
  );
 });

 lines.push('END:VCALENDAR');

 let blob = new Blob([lines.join('\r\n')], { type: 'text/calendar;charset=utf-8' });
 let link = document.createElement('a');
 link.href = URL.createObjectURL(blob);
 link.download = `${activeName.replace(/[^a-z0-9]/gi, '_').toLowerCase()}_manual.ics`;
 link.click();
}

refreshListings();
render();
</script>
</body>
</html>[preview.html](https://github.com/user-attachments/files/31557178/preview.html)
