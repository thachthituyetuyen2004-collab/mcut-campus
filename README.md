# mcut-campus
Mcut campus
campus-sim/
├─ index.html
├─ app.js
├─ style.css
├─ manifest.json
├─ sw.js
├─ campus.jpg        <-- ảnh bản đồ của bạn (đổi tên thành campus.jpg)
└─ data/
   ├─ nodes.json     (tùy chọn; app có thể lưu local)
   └─ edges.json
<!doctype html>
<html>
<head>
  <meta charset="utf-8"/>
  <meta name="viewport" content="width=device-width,initial-scale=1"/>
  <title>Campus Nav Simulator</title>
  <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css"/>
  <link rel="manifest" href="manifest.json">
  <link rel="stylesheet" href="style.css">
</head>
<body>
  <div id="controls">
    <button id="mode-add-node">Add Node</button>
    <button id="mode-add-edge">Add Edge</button>
    <button id="mode-select">Select / Move</button>
    <button id="save-json">Save JSON</button>
    <input id="load-file" type="file" accept=".json"/>
    <button id="compute-route">Compute Route</button>
    <button id="playback">Playback Sim</button>
    <button id="clear">Clear</button>
    <div style="font-size:12px;margin-top:6px">Lưu ý: chỉnh IMAGE_WIDTH, IMAGE_HEIGHT trong app.js nếu cần</div>
  </div>
  <div id="map"></div>

  <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
  <script src="app.js"></script>
</body>
</html>
html,body,#map{height:100%;margin:0;padding:0}
#controls{
  position: absolute; z-index:1000; left:8px; top:8px;
  background: rgba(255,255,255,0.96); padding:8px; border-radius:6px;
  box-shadow: 0 1px 6px rgba(0,0,0,0.2); max-width:260px;
}
#controls button, #controls input{margin:4px 2px; font-size:13px}
{
  "name": "Campus Nav Simulator",
  "short_name": "CampusSim",
  "start_url": ".",
  "display": "standalone",
  "background_color": "#ffffff",
  "icons":[
    { "src": "campus.jpg", "sizes":"192x192", "type":"image/jpeg" }
  ]
}
const CACHE = 'campus-sim-v1';
self.addEventListener('install', e => {
  e.waitUntil(caches.open(CACHE).then(c => c.addAll(['/', '/index.html', '/app.js', '/style.css', '/campus.jpg'])));
});
self.addEventListener('fetch', e => {
  e.respondWith(caches.match(e.request).then(r => r || fetch(e.request)));
});
// app.js - Campus Nav Simulator (Leaflet + CRS.Simple)
// ====== CONFIG ======
const IMAGE_URL = 'campus.jpg'; // file ảnh trong repo
const IMAGE_WIDTH = 2000;  // <-- chỉnh theo kích thước pixel của campus.jpg
const IMAGE_HEIGHT = 1200; // <-- chỉnh theo kích thước pixel của campus.jpg

// ====== STATE ======
let map, imageLayer, bounds;
let mode = 'select';
let nodes = [];
let edges = [];
let markers = {};
let selectedNodeForEdge = null;
let routeLayer = null;
let userMarker = null;

function ds(id){ return document.getElementById(id); }
function toXY(latlng){ return {x: latlng.lng, y: latlng.lat}; }
function uid(){ return Date.now().toString(36) + Math.floor(Math.random()*1000); }

function init(){
  map = L.map('map', { crs: L.CRS.Simple, minZoom: -5 });
  bounds = [[0,0],[IMAGE_HEIGHT, IMAGE_WIDTH]];
  imageLayer = L.imageOverlay(IMAGE_URL, bounds).addTo(map);
  map.fitBounds(bounds);
  map.on('click', onMapClick);

  ds('mode-add-node').onclick = ()=> { mode='add-node'; alert('Click trên map để thêm node'); };
  ds('mode-add-edge').onclick = ()=> { mode='add-edge'; selectedNodeForEdge=null; alert('Chọn 2 node để tạo edge'); };
  ds('mode-select').onclick = ()=> { mode='select'; };
  ds('save-json').onclick = saveJSON;
  ds('load-file').addEventListener('change', loadFromFile);
  ds('compute-route').onclick = computeRouteForDemo;
  ds('playback').onclick = startPlayback;
  ds('clear').onclick = ()=>{ clearAll(); };

  userMarker = L.circleMarker([10,10], {radius:6, color:'red'}).addTo(map).setOpacity(0);

  const saved = localStorage.getItem('campus_nodes');
  if(saved){
    try{ const parsed = JSON.parse(saved); nodes=parsed.nodes||[]; edges=parsed.edges||[]; rebuildMarkers(); }catch(e){}
  }
}

function onMapClick(e){
  const xy = toXY(e.latlng);
  if(mode === 'add-node'){
    const name = prompt('Tên node (ví dụ: Bldg 1):', '');
    if(!name) return;
    const id = uid();
    const node = {id, name, x: xy.x, y: xy.y};
    nodes.push(node);
    addMarker(node);
    persist();
  } else if(mode === 'add-edge'){
    const nearest = findNearestNode(xy, 40);
    if(!nearest){ alert('Không tìm node gần điểm click'); return; }
    if(!selectedNodeForEdge){
      selectedNodeForEdge = nearest;
      alert('Chọn node thứ hai để hoàn thành edge');
    } else {
      if(selectedNodeForEdge.id === nearest.id){ selectedNodeForEdge=null; return; }
      const dist = euclidean(selectedNodeForEdge, nearest);
      edges.push({from: selectedNodeForEdge.id, to: nearest.id, cost: dist});
      edges.push({from: nearest.id, to: selectedNodeForEdge.id, cost: dist});
      drawEdge(selectedNodeForEdge, nearest);
      selectedNodeForEdge = null;
      persist();
    }
  }
}

function addMarker(node){
  const m = L.marker([node.y, node.x], {draggable:true}).addTo(map);
  m.bindTooltip(node.name, {permanent:true, direction:'top'});
  m.on('dragend', (ev)=>{
    const p = ev.target.getLatLng();
    node.x = p.lng; node.y = p.lat;
    persist();
    redrawEdges();
  });
  m.on('click', ()=>{
    if(mode === 'select'){
      alert(`Node: ${node.name}\nID: ${node.id}\ncoords: ${Math.round(node.x)},${Math.round(node.y)}`);
    }
  });
  markers[node.id] = m;
}

function rebuildMarkers(){
  for(const k in markers) map.removeLayer(markers[k]);
  markers = {};
  nodes.forEach(n=> addMarker(n));
  redrawEdges();
}

function drawEdge(n1, n2){
  L.polyline([[n1.y,n1.x],[n2.y,n2.x]], {color:'#888'}).addTo(map);
}
function redrawEdges(){
  map.eachLayer(layer=>{
    if(layer === imageLayer) return;
    if(layer === userMarker) return;
    if(Object.values(markers).includes(layer)) return;
    if(routeLayer && layer === routeLayer) return;
    map.removeLayer(layer);
  });
  edges.forEach(e=>{
    const a = nodes.find(n=> n.id===e.from);
    const b = nodes.find(n=> n.id===e.to);
    if(a && b) drawEdge(a,b);
  });
}

function findNearestNode(xy, maxDistPx=40){
  let best=null, bestd=Infinity;
  nodes.forEach(n=>{
    const d = Math.hypot(n.x - xy.x, n.y - xy.y);
    if(d < bestd && d <= maxDistPx){ bestd = d; best = n; }
  });
  return best;
}

function buildGraph(){
  const g = {};
  edges.forEach(e=>{
    if(!g[e.from]) g[e.from]=[];
    g[e.from].push({to: e.to, cost: e.cost});
  });
  return g;
}
function euclidean(a,b){
  return Math.hypot(a.x-b.x, a.y-b.y);
}

class PQ {
  constructor(){ this.arr=[]; }
  push(item,priority){ this.arr.push({item,priority}); }
  pop(){
    let bestI=0;
    for(let i=1;i<this.arr.length;i++) if(this.arr[i].priority < this.arr[bestI].priority) bestI=i;
    const v = this.arr.splice(bestI,1)[0];
    return v.item;
  }
  isEmpty(){ return this.arr.length===0; }
}

function aStar(startId, goalId){
  const graph = buildGraph();
  const open = new PQ();
  const cameFrom = {};
  const gScore = {};
  nodes.forEach(n=> gScore[n.id]=Infinity);
  gScore[startId] = 0;
  const startNode = nodes.find(n=> n.id===startId);
  const goalNode = nodes.find(n=> n.id===goalId);
  open.push(startId, euclidean(startNode, goalNode));
  while(!open.isEmpty()){
    const current = open.pop();
    if(current === goalId){
      const path = [];
      let cur = current;
      while(cur){ path.push(cur); cur = cameFrom[cur]; }
      return path.reverse();
    }
    const neighbors = graph[current] || [];
    neighbors.forEach(nb=>{
      const tentative = gScore[current] + nb.cost;
      if(tentative < (gScore[nb.to] ?? Infinity)){
        cameFrom[nb.to] = current;
        gScore[nb.to] = tentative;
        const nodeTo = nodes.find(n=>n.id===nb.to);
        const priority = tentative + euclidean(nodeTo, goalNode);
        open.push(nb.to, priority);
      }
    });
  }
  return null;
}

function showRoute(nodeIdPath){
  if(routeLayer) map.removeLayer(routeLayer);
  const latlngs = nodeIdPath.map(id=>{
    const n = nodes.find(x=> x.id===id);
    return [n.y, n.x];
  });
  routeLayer = L.polyline(latlngs, {color:'blue', weight:4}).addTo(map);
}

function saveJSON(){
  const content = {nodes, edges};
  const blob = new Blob([JSON.stringify(content, null,2)], {type:'application/json'});
  const url = URL.createObjectURL(blob);
  const a = document.createElement('a');
  a.href = url; a.download = 'campus-data.json'; a.click();
  URL.revokeObjectURL(url);
  localStorage.setItem('campus_nodes', JSON.stringify(content));
  alert('Saved to campus-data.json and localStorage');
}

function loadFromFile(ev){
  const f = ev.target.files[0];
  if(!f) return;
  const reader = new FileReader();
  reader.onload = e=>{
    try{
      const obj = JSON.parse(e.target.result);
      nodes = obj.nodes||[]; edges = obj.edges||[];
      rebuildMarkers();
      persist();
      alert('Loaded JSON');
    }catch(err){ alert('Invalid JSON'); }
  };
  reader.readAsText(f);
}

function persist(){ localStorage.setItem('campus_nodes', JSON.stringify({nodes,edges})); }

function computeRouteForDemo(){
  if(nodes.length < 2){ alert('Cần ít nhất 2 node'); return; }
  const names = nodes.map(n=> `${n.id}:${n.name}`).join('\n');
  const fromId = prompt('Nhập id node bắt đầu (ví dụ 1):\n' + names);
  const toId = prompt('Nhập id node đích (ví dụ 2):\n' + names);
  if(!fromId || !toId){ alert('Canceled'); return; }
  const path = aStar(fromId, toId);
  if(!path){ alert('Không tìm đường (graph không liên thông)'); return; }
  showRoute(path);
  const start = nodes.find(n=>n.id===path[0]);
  userMarker.setLatLng([start.y, start.x]).setOpacity(1);
}

let playbackTimer = null;
function startPlayback(){
  if(!routeLayer){ alert('Chạy Compute Route trước'); return; }
  const latlngs = routeLayer.getLatLngs();
  let i=0;
  if(playbackTimer) clearInterval(playbackTimer);
  playbackTimer = setInterval(()=>{
    if(i >= latlngs.length){ clearInterval(playbackTimer); return; }
    userMarker.setLatLng(latlngs[i]).setOpacity(1);
    i++;
  }, 700);
}

function clearAll(){
  nodes = []; edges = [];
  for(const k in markers){ map.removeLayer(markers[k]); }
  markers = {};
  map.eachLayer(layer=>{
    if(layer===imageLayer) return;
    map.removeLayer(layer);
  });
  persist();
}

init();
[
  {"id":"n1","name":"Bldg A","x":300,"y":200},
  {"id":"n2","name":"Bldg B","x":600,"y":220},
  {"id":"n3","name":"Library","x":900,"y":300}
]
[
  {"from":"n1","to":"n2","cost":300},
  {"from":"n2","to":"n1","cost":300},
  {"from":"n2","to":"n3","cost":320},
  {"from":"n3","to":"n2","cost":320}
]
