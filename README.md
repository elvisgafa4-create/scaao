
<!DOCTYPE html>
<html lang="fr">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>Générateur de cadre — Campagne</title>
<style>
  :root{
    --bg:#0f1115;
    --panel:#1a1d24;
    --accent:#c1652f; /* terracotta */
    --text:#f2f0ec;
    --muted:#9a9a9a;
  }
  *{box-sizing:border-box;}
  body{
    margin:0;
    font-family:'Segoe UI', Roboto, Arial, sans-serif;
    background:var(--bg);
    color:var(--text);
    display:flex;
    flex-direction:column;
    align-items:center;
    padding:32px 16px 64px;
  }
  h1{
    font-size:1.4rem;
    margin-bottom:4px;
    text-align:center;
  }
  p.sub{
    color:var(--muted);
    margin-top:0;
    margin-bottom:24px;
    text-align:center;
    font-size:0.9rem;
  }
  .layout{
    display:flex;
    gap:24px;
    flex-wrap:wrap;
    justify-content:center;
    width:100%;
    max-width:900px;
  }
  .panel{
    background:var(--panel);
    border-radius:14px;
    padding:20px;
    flex:1 1 260px;
  }
  #canvasWrap{
    position:relative;
    width:400px;
    height:400px;
    max-width:90vw;
    max-height:90vw;
    margin:0 auto;
    border-radius:12px;
    overflow:hidden;
    background:repeating-conic-gradient(#2a2d35 0% 25%, #22242b 0% 50%) 50% / 20px 20px;
    touch-action:none;
    cursor:grab;
  }
  #canvasWrap:active{cursor:grabbing;}
  canvas{
    position:absolute;
    top:0;left:0;
    width:100%;
    height:100%;
  }
  label{
    display:block;
    font-size:0.85rem;
    margin:14px 0 6px;
    color:var(--muted);
  }
  input[type="file"], input[type="range"]{
    width:100%;
  }
  input[type="file"]{
    color:var(--text);
    font-size:0.85rem;
  }
  button{
    width:100%;
    padding:12px;
    margin-top:18px;
    border:none;
    border-radius:8px;
    background:var(--accent);
    color:#fff;
    font-weight:600;
    font-size:0.95rem;
    cursor:pointer;
    transition:opacity .2s;
  }
  button:hover{opacity:0.9;}
  button:disabled{
    opacity:0.4;
    cursor:not-allowed;
  }
  .hint{
    font-size:0.78rem;
    color:var(--muted);
    margin-top:10px;
    line-height:1.4;
  }
  code{
    background:#000;
    padding:2px 6px;
    border-radius:4px;
    font-size:0.78rem;
  }
</style>
</head>
<body>

  <h1>Générateur de cadre de campagne</h1>
  <p class="sub">Ajoute ta photo, positionne-la, télécharge l'image avec le cadre superposé.</p>

  <div class="layout">

    <div class="panel">
      <div id="canvasWrap">
        <canvas id="canvas" width="800" height="800"></canvas>
      </div>
      <p class="hint">Glisse l'image pour la repositionner sous le cadre.</p>
    </div>

    <div class="panel">
      <label for="photoInput">1. Ta photo</label>
      <input type="file" id="photoInput" accept="image/*">

      <label for="zoom">2. Zoom</label>
      <input type="range" id="zoom" min="0.5" max="3" step="0.01" value="1">

      <label for="frameInput">3. Cadre de la campagne (PNG transparent, optionnel si déjà dans le dossier sous frame.png)</label>
      <input type="file" id="frameInput" accept="image/png">

      <button id="downloadBtn" disabled>Télécharger l'image finale</button>

      <p class="hint">
        Place un fichier <code>frame.png</code> (transparent au centre) à côté de ce fichier HTML
        pour qu'il se charge automatiquement, ou choisis-le manuellement au point 3.
        Le cadre doit être un PNG carré avec un trou transparent au milieu.
      </p>
    </div>

  </div>

<script>
const canvas = document.getElementById('canvas');
const ctx = canvas.getContext('2d');
const canvasWrap = document.getElementById('canvasWrap');
const photoInput = document.getElementById('photoInput');
const frameInput = document.getElementById('frameInput');
const zoomSlider = document.getElementById('zoom');
const downloadBtn = document.getElementById('downloadBtn');

const SIZE = canvas.width; // 800x800 export

let userImg = null;
let frameImg = new Image();
let frameLoaded = false;

// offset (en pixels, repère du canvas 800x800) et zoom du user photo
let offsetX = 0, offsetY = 0, scale = 1;
let dragging = false, lastX = 0, lastY = 0;

// Tente de charger frame.png automatiquement à côté du fichier
frameImg.onload = () => { frameLoaded = true; draw(); };
frameImg.onerror = () => { frameLoaded = false; };
frameImg.src = 'frame.png';

frameInput.addEventListener('change', (e) => {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (ev) => {
    frameImg = new Image();
    frameImg.onload = () => { frameLoaded = true; draw(); };
    frameImg.src = ev.target.result;
  };
  reader.readAsDataURL(file);
});

photoInput.addEventListener('change', (e) => {
  const file = e.target.files[0];
  if (!file) return;
  const reader = new FileReader();
  reader.onload = (ev) => {
    userImg = new Image();
    userImg.onload = () => {
      // centre + couvre le canvas par défaut
      const ratio = Math.max(SIZE / userImg.width, SIZE / userImg.height);
      scale = ratio;
      zoomSlider.value = 1;
      offsetX = (SIZE - userImg.width * scale) / 2;
      offsetY = (SIZE - userImg.height * scale) / 2;
      downloadBtn.disabled = false;
      draw();
    };
    userImg.src = ev.target.result;
  };
  reader.readAsDataURL(file);
});

zoomSlider.addEventListener('input', () => {
  if (!userImg) return;
  const baseRatio = Math.max(SIZE / userImg.width, SIZE / userImg.height);
  const centerX = SIZE / 2, centerY = SIZE / 2;
  // point sous le centre du canvas dans l'espace image avant zoom
  const imgCenterX = (centerX - offsetX) / scale;
  const imgCenterY = (centerY - offsetY) / scale;

  scale = baseRatio * parseFloat(zoomSlider.value);

  offsetX = centerX - imgCenterX * scale;
  offsetY = centerY - imgCenterY * scale;
  draw();
});

// Drag (souris)
canvasWrap.addEventListener('mousedown', (e) => {
  if (!userImg) return;
  dragging = true;
  lastX = e.clientX;
  lastY = e.clientY;
});
window.addEventListener('mousemove', (e) => {
  if (!dragging) return;
  const rect = canvasWrap.getBoundingClientRect();
  const dx = (e.clientX - lastX) * (SIZE / rect.width);
  const dy = (e.clientY - lastY) * (SIZE / rect.height);
  offsetX += dx;
  offsetY += dy;
  lastX = e.clientX;
  lastY = e.clientY;
  draw();
});
window.addEventListener('mouseup', () => dragging = false);

// Drag (tactile)
canvasWrap.addEventListener('touchstart', (e) => {
  if (!userImg) return;
  dragging = true;
  lastX = e.touches[0].clientX;
  lastY = e.touches[0].clientY;
}, {passive:true});
canvasWrap.addEventListener('touchmove', (e) => {
  if (!dragging) return;
  const rect = canvasWrap.getBoundingClientRect();
  const dx = (e.touches[0].clientX - lastX) * (SIZE / rect.width);
  const dy = (e.touches[0].clientY - lastY) * (SIZE / rect.height);
  offsetX += dx;
  offsetY += dy;
  lastX = e.touches[0].clientX;
  lastY = e.touches[0].clientY;
  draw();
}, {passive:true});
window.addEventListener('touchend', () => dragging = false);

function draw(){
  ctx.clearRect(0, 0, SIZE, SIZE);
  if (userImg){
    ctx.drawImage(userImg, offsetX, offsetY, userImg.width * scale, userImg.height * scale);
  }
  if (frameLoaded){
    ctx.drawImage(frameImg, 0, 0, SIZE, SIZE);
  }
}

downloadBtn.addEventListener('click', () => {
  const link = document.createElement('a');
  link.download = 'ma-photo-campagne.png';
  link.href = canvas.toDataURL('image/png');
  link.click();
});

draw();
</script>

</body>
</html>
