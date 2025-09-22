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
