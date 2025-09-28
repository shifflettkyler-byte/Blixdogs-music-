<!doctype html>
<html lang="en">
<head>
<meta charset="utf-8" />
<meta name="viewport" content="width=device-width,initial-scale=1" />
<title>Music Site — Live Background + Player</title>
<style>
  /* layout */
  :root{
    --accent: 255,180,80;
    --bg-dark: #050510;
    --panel-bg: rgba(0,0,0,0.45);
  }
  html,body{height:100%;margin:0;font-family:system-ui,Segoe UI,Roboto,Helvetica,Arial;background:var(--bg-dark);color:#fff}
  .wrap{position:relative;height:100vh;overflow:hidden;display:flex;align-items:center;justify-content:center;padding:24px;box-sizing:border-box}
  /* canvas fills the screen */
  canvas.bg{position:absolute;inset:0;z-index:0}
  /* panels */
  .ui{position:relative;z-index:5;width:min(980px,95%);display:grid;grid-template-columns:1fr 360px;gap:20px}
  @media (max-width:880px){.ui{grid-template-columns:1fr;}}
  .left{
    background:linear-gradient(180deg, rgba(255,255,255,0.02), rgba(255,255,255,0.01));
    border-radius:12px;padding:26px;backdrop-filter: blur(6px);box-shadow:0 6px 30px rgba(0,0,0,0.6);
  }
  .title{font-size:clamp(20px,3.2vw,28px);margin:0 0 8px}
  .desc{opacity:0.85;margin:0 0 16px}
  /* visualizer canvas inside left panel */
  .vis-wrap{height:220px;border-radius:8px;overflow:hidden;background:linear-gradient(180deg, rgba(0,0,0,0.25), rgba(255,255,255,0.02));display:flex;align-items:center;justify-content:center}
  /* playlist panel */
  .right{background:var(--panel-bg);padding:18px;border-radius:12px;display:flex;flex-direction:column;gap:12px}
  .controls{display:flex;gap:8px;align-items:center}
  button{background:rgba(255,255,255,0.06);border:1px solid rgba(255,255,255,0.06);color:white;padding:8px 12px;border-radius:8px;cursor:pointer}
  button:hover{transform:translateY(-1px)}
  .track-list{overflow:auto;max-height:56vh;padding-right:6px}
  .track{padding:8px;border-radius:8px;margin-bottom:6px;display:flex;justify-content:space-between;align-items:center;gap:8px}
  .track:hover{background:rgba(255,255,255,0.02)}
  .track.active{background:rgba(255,255,255,0.04);box-shadow:inset 0 0 0 1px rgba(255,255,255,0.02)}
  .meta{font-size:14px;opacity:0.95}
  .muted-note{opacity:0.7;font-size:13px}
  input[type="range"]{width:110px}
  .small{font-size:13px;opacity:0.9}
  footer{margin-top:6px;font-size:12px;opacity:0.8}
</style>
</head>
<body>
<div class="wrap">
  <canvas class="bg" id="bgCanvas"></canvas>

  <div class="ui" role="main" aria-live="polite">
    <div class="left">
      <h1 class="title">Live Music Background</h1>
      <p class="desc">Background visuals react to the music. Click Play — browsers often block autoplay with sound, so user action is required.</p>

      <div class="vis-wrap">
        <canvas id="visCanvas" width="1200" height="400" style="width:100%;height:100%"></canvas>
      </div>

      <p class="small" style="margin-top:12px">Tip: host your audio files on the same domain (or allow CORS), then replace the entries in the playlist below.</p>
    </div>

    <aside class="right" aria-label="Music player">
      <div style="display:flex;justify-content:space-between;align-items:center">
        <div>
          <div class="muted-note">Now playing</div>
          <div id="nowTitle" style="font-weight:700">—</div>
        </div>
        <div class="controls">
          <button id="prevBtn" title="Previous">⟵</button>
          <button id="playBtn" title="Play / Pause">Play</button>
          <button id="nextBtn" title="Next">⟶</button>
        </div>
      </div>

      <div style="display:flex;gap:8px;align-items:center;margin-top:10px">
        <label class="small">Volume</label>
        <input id="vol" type="range" min="0" max="1" step="0.01" value="0.8" />
        <button id="shuffle" title="Shuffle">🔀</button>
      </div>

      <div style="margin-top:10px" class="small">Playlist</div>
      <div class="track-list" id="tracks"></div>

      <footer>
        <div class="small">Replace the sample tracks in the script with your own hosted MP3 files or URLs. Autoplay may be blocked; users should press Play.</div>
      </footer>
    </aside>
  </div>
</div>

<!-- hidden audio element -->
<audio id="audio" crossorigin="anonymous"></audio>

<script>
/*
  HOW TO ADD YOUR MUSIC:
  - Edit the `playlist` array below.
  - Each item: { title: "Song name", src: "https://yourhost.com/path/to/song.mp3" }
  - You can host files on the same server or use a public CDN that allows audio access (CORS).
  - Browsers often block autoplay with audio: user must click Play.
*/

/* =========== Playlist (replace these with your tracks) =========== */
const playlist = [
  { title: "Sample Track 1 — Replace me", src: "https://cdn.jsdelivr.net/gh/anars/blank-audio/250-milliseconds.mp3" },
  { title: "Sample Track 2 — Replace me", src: "https://cdn.jsdelivr.net/gh/anars/blank-audio/500-milliseconds.mp3" },
  /* Example of adding more:
     { title: "My Banger", src: "https://your-domain.com/music/my-banger.mp3" }
  */
];
/* ================================================================= */

const audio = document.getElementById('audio');
let current = 0;
let isPlaying = false;
let shuffleMode = false;

const nowTitle = document.getElementById('nowTitle');
const playBtn = document.getElementById('playBtn');
const prevBtn = document.getElementById('prevBtn');
const nextBtn = document.getElementById('nextBtn');
const volRange = document.getElementById('vol');
const shuffleBtn = document.getElementById('shuffle');
const tracksEl = document.getElementById('tracks');

function renderPlaylist(){
  tracksEl.innerHTML = '';
  playlist.forEach((t, i) => {
    const div = document.createElement('div');
    div.className = 'track' + (i===current ? ' active':'');
    div.innerHTML = `<div class="meta">${t.title}</div><div style="display:flex;gap:8px"><button data-i="${i}" class="play-one">▶</button></div>`;
    div.querySelector('.play-one').addEventListener('click', () => { playIndex(i); });
    tracksEl.appendChild(div);
  });
}
function updateNow(){
  nowTitle.textContent = playlist[current] ? playlist[current].title : '—';
  Array.from(tracksEl.children).forEach((el, idx) => el.classList.toggle('active', idx===current));
}

function playIndex(i){
  if (i<0 || i>=playlist.length) return;
  current = i;
  audio.src = playlist[current].src;
  audio.play().catch(err=>console.warn('Play blocked',err));
  isPlaying = true;
  playBtn.textContent = 'Pause';
  updateNow();
  resumeAudioContext();
}

function prev(){
  if (shuffleMode) {
    playIndex(Math.floor(Math.random()*playlist.length));
    return;
  }
  current = (current-1 + playlist.length) % playlist.length;
  playIndex(current);
}
function next(){
  if (shuffleMode) {
    playIndex(Math.floor(Math.random()*playlist.length));
    return;
  }
  current = (current+1) % playlist.length;
  playIndex(current);
}

playBtn.addEventListener('click', ()=>{
  if (!isPlaying){
    if (!audio.src) playIndex(current);
    else audio.play().catch(err=>console.warn('Play blocked',err));
    isPlaying = true;
    playBtn.textContent = 'Pause';
    resumeAudioContext();
  } else {
    audio.pause();
    isPlaying = false;
    playBtn.textContent = 'Play';
  }
});
prevBtn.addEventListener('click', prev);
nextBtn.addEventListener('click', next);
volRange.addEventListener('input', e => { audio.volume = e.target.value; });
shuffleBtn.addEventListener('click', ()=>{ shuffleMode = !shuffleMode; shuffleBtn.style.opacity = shuffleMode?1:0.7; });

audio.addEventListener('ended', next);
audio.addEventListener('play', ()=>{ isPlaying = true; playBtn.textContent='Pause' });
audio.addEventListener('pause', ()=>{ isPlaying = false; playBtn.textContent='Play' });

renderPlaylist();
updateNow();
audio.volume = Number(volRange.value);

/* =========== Audio visualizer & particles =========== */
const visCanvas = document.getElementById('visCanvas');
const vctx = visCanvas.getContext('2d');

let audioCtx, analyser, sourceNode, dataArray, bufferLength;
let animationId;
function setupAudioNodes(){
  if (audioCtx) return;
  audioCtx = new (window.AudioContext || window.webkitAudioContext)();
  analyser = audioCtx.createAnalyser();
  analyser.fftSize = 2048;
  bufferLength = analyser.frequencyBinCount;
  dataArray = new Uint8Array(bufferLength);

  sourceNode = audioCtx.createMediaElementSource(audio);
  sourceNode.connect(analyser);
  analyser.connect(audioCtx.destination);
}
function resumeAudioContext(){
  try{
    if (!audioCtx) setupAudioNodes();
    if (audioCtx.state === 'suspended') audioCtx.resume();
    if (!animationId) draw();
  }catch(e){ console.warn('AudioContext error', e) }
}

/* simple particle system that reacts to bass */
const particles = [];
function spawnParticle(x,y,size,life,vel){
  particles.push({x,y,size,life,age:0,vel:vel || (Math.random()*2-1)});
}
function draw(){
  if (!analyser) return;
  analyser.getByteFrequencyData(dataArray);

  const width = visCanvas.width = visCanvas.clientWidth * devicePixelRatio;
  const height = visCanvas.height = visCanvas.clientHeight * devicePixelRatio;

  vctx.clearRect(0,0,width,height);
  // draw bars
  const barCount = 80;
  const slice = Math.floor(bufferLength / barCount);
  const barW = width / barCount;
  for (let i=0;i<barCount;i++){
    let sum=0;
    for (let j=0;j<slice;j++) sum += dataArray[i*slice + j];
    const val = sum / slice;
    const h = (val/255)*height*0.9;
    const x = i*barW;
    // color based on index
    const hue = Math.floor(200 - (i/barCount)*120);
    vctx.fillStyle = `rgba(${hue},200,255,${0.9})`;
    vctx.fillRect(x, height - h, barW*0.8, h);
  }

  // spawn particles based on low-frequency energy
  let bass = 0;
  for (let i=0;i<10;i++) bass += dataArray[i];
  bass = bass / (10*255); // 0..1
  const spawnCount = Math.round(bass*6);
  for (let i=0;i<spawnCount;i++){
    spawnParticle(Math.random()*width, height - 20, 6 + Math.random()*16, 40 + Math.random()*80, (Math.random()*2-1)*1.5);
  }

  // update and draw particles
  for (let i=particles.length-1;i>=0;i--){
    const p = particles[i];
    p.age++;
    p.x += p.vel*2;
    p.y -= 1 + (p.size/20);
    const lifeRatio = 1 - (p.age/p.life);
    vctx.beginPath();
    vctx.fillStyle = `rgba(255, ${120 + Math.round(lifeRatio*120)}, ${Math.round(50 + lifeRatio*120)}, ${lifeRatio})`;
    vctx.arc(p.x, p.y, Math.max(0, p.size*lifeRatio), 0, Math.PI*2);
    vctx.fill();
    if (p.age > p.life) particles.splice(i,1);
  }

  animationId = requestAnimationFrame(draw);
}

/* =========== Fullscreen background animated gradient & particles =========== */
const bgCanvas = document.getElementById('bgCanvas');
const bg = bgCanvas.getContext('2d');
let bgParticles = [];
function resizeBg(){
  bgCanvas.width = innerWidth * devicePixelRatio;
  bgCanvas.height = innerHeight * devicePixelRatio;
  bgCanvas.style.width = innerWidth + 'px';
  bgCanvas.style.height = innerHeight + 'px';
}
function spawnBgParticle(){
  bgParticles.push({
    x: Math.random()*bgCanvas.width,
    y: Math.random()*bgCanvas.height,
    r: 10 + Math.random()*80,
    v: (Math.random()*0.6)+0.05,
    hue: Math.random()*360
  });
}
for(let i=0;i<22;i++) spawnBgParticle();
resizeBg();
window.addEventListener('resize', resizeBg);

let hueShift = 0;
function drawBg(){
  const w = bgCanvas.width, h = bgCanvas.height;
  hueShift = (hueShift + 0.08) % 360;
  // animated radial gradient overlay
  const g = bg.createLinearGradient(0,0,w,h);
  g.addColorStop(0, `hsla(${(hueShift)%360},60%,6%,0.75)`);
  g.addColorStop(0.45, `hsla(${(hueShift+40)%360},65%,8%,0.55)`);
  g.addColorStop(1, `hsla(${(hueShift+90)%360},65%,10%,0.75)`);
  bg.fillStyle = g;
  bg.fillRect(0,0,w,h);

  // draw soft glow blobs (background particles)
  for (let p of bgParticles){
    p.y -= p.v;
    if (p.y + p.r < 0) p.y = h + p.r;
    const rg = bg.createRadialGradient(p.x, p.y, 0, p.x, p.y, p.r*2);
    rg.addColorStop(0, `hsla(${p.hue},80%,60%,${0.08})`);
    rg.addColorStop(1, `hsla(${p.hue},80%,60%,0)`);
    bg.fillStyle = rg;
    bg.fillRect(p.x - p.r*2, p.y - p.r*2, p.r*4, p.r*4);
  }

  requestAnimationFrame(drawBg);
}
drawBg();

/* =========== Accessibility & fallback notes =========== */
/*
 - Browsers often block autoplay of audio with sound. The user should click Play.
 - On many mobile devices, WebAudio analyzers need user gesture to start.
 - If you host audio on another domain, make sure CORS is enabled for audio files or MediaElementSource/Analyser will fail.
 - If you want to embed Spotify/Apple Music: those embeds won't expose raw audio to the WebAudio API for visualization. Use hosted mp3s or an audio provider that allows direct audio streaming.
*/

/* initialize but do not start audioContext until user interacts */
document.addEventListener('click', resumeAudioContext, {once:true});
document.addEventListener('keydown', resumeAudioContext, {once:true});

/* On load, don't autoplay — user must press play for best cross-browser results */
</script>
</body>
</html>
