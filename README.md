<!doctype html>
<html lang="id">
<head>
  <meta charset="utf-8" />
  <meta name="viewport" content="width=device-width,initial-scale=1" />
  <title>Simulasi Interferensi Gelombang (Babylon.js)</title>
  <style>
    html,body { height:100%; margin:0; font-family: Inter, Roboto, system-ui, sans-serif; background:#0f172a; color:#e6eef8; }
    #ui {
      position: absolute; left: 12px; top: 12px; z-index: 20;
      background: rgba(3,7,18,0.75); padding:12px; border-radius:10px;
      box-shadow: 0 6px 24px rgba(2,6,23,0.6); width:360px;
      backdrop-filter: blur(6px);
    }
    h1 { margin:0 0 8px 0; font-size:16px; color:#e6eef8; }
    label { display:block; font-size:12px; margin-top:10px; color:#cfe3ff; }
    input[type="range"] { width:100%; }
    .row { display:flex; gap:8px; align-items:center; margin-top:8px; }
    .small { font-size:12px; color:#9fb7df; }
    button {
      margin-top:12px; padding:8px 10px; border-radius:8px; border:none; cursor:pointer;
      background: linear-gradient(180deg,#2563eb,#1e40af); color:white; font-weight:600;
    }
    #canvasContainer { width:100%; height:100%; }
    .legend { font-size:12px; margin-top:8px; }
    .value { min-width:56px; text-align:right; color:#dbeafe; font-weight:700; }
    .footerNote { font-size:11px; color:#9fb7df; margin-top:8px; }
    a.link { color:#93c5fd; text-decoration:underline; }
  </style>
</head>
<body>
  <div id="ui" role="region" aria-label="Kontrol simulasi">
    <h1>Simulasi Interferensi Gelombang — XR/AI Project</h1>
    <div class="small">Ubah parameter dan tekan <strong>Analisis AI</strong> untuk saran.</div>

    <label>Amplitudo 1: <span id="A1val" class="value">1.0</span></label>
    <input id="A1" type="range" min="0" max="2" step="0.01" value="1.0">

    <label>Frekuensi 1: <span id="f1val" class="value">2.0</span></label>
    <input id="f1" type="range" min="0.2" max="6" step="0.01" value="2.0">

    <label>Fase 1 (deg): <span id="p1val" class="value">0</span></label>
    <input id="p1" type="range" min="0" max="360" step="1" value="0">

    <hr style="border-color:#10203b;" />

    <label>Amplitudo 2: <span id="A2val" class="value">1.0</span></label>
    <input id="A2" type="range" min="0" max="2" step="0.01" value="1.0">

    <label>Frekuensi 2: <span id="f2val" class="value">2.0</span></label>
    <input id="f2" type="range" min="0.2" max="6" step="0.01" value="2.0">

    <label>Fase 2 (deg): <span id="p2val" class="value">180</span></label>
    <input id="p2" type="range" min="0" max="360" step="1" value="180">

    <div class="row">
      <button id="aiAnalyze">Analisis AI</button>
      <button id="toggleAnimate" style="background:linear-gradient(180deg,#10b981,#065f46);">Toggle Anim</button>
    </div>

    <div class="legend">
      <div><strong>Legend:</strong></div>
      <div class="small">Gelombang 1 — biru &nbsp; | &nbsp; Gelombang 2 — oranye &nbsp; | &nbsp; Hasil — putih</div>
      <div class="footerNote">Buka di browser modern. Untuk XR/VR, gunakan perangkat & browser yang mendukung WebXR.</div>
    </div>
  </div>

  <div id="canvasContainer">
    <canvas id="renderCanvas" touch-action="none" style="width:100%; height:100%;"></canvas>
  </div>

  <!-- Babylon.js CDN -->
  <script src="https://cdn.babylonjs.com/babylon.js"></script>
  <script src="https://cdn.babylonjs.com/gui/babylon.gui.min.js"></script>

  <script>
  (function(){
    const canvas = document.getElementById('renderCanvas');
    const engine = new BABYLON.Engine(canvas, true, {preserveDrawingBuffer:true, stencil:true});
    const scene = new BABYLON.Scene(engine);
    scene.clearColor = new BABYLON.Color4(0.04, 0.06, 0.14, 1);

    // Camera
    const camera = new BABYLON.ArcRotateCamera("cam", -Math.PI/2, Math.PI/3.5, 14, new BABYLON.Vector3(0,0,0), scene);
    camera.attachControl(canvas, true);
    camera.lowerRadiusLimit = 4;
    camera.upperRadiusLimit = 40;

    // Light
    const light = new BABYLON.HemisphericLight("hemi", new BABYLON.Vector3(0,1,0), scene);
    light.intensity = 0.9;

    // Ground grid (visual reference)
    const grid = BABYLON.MeshBuilder.CreateGround("grid", {width:20, height:6, subdivisions:40}, scene);
    grid.position.y = -3.5;
    grid.isPickable = false;
    grid.visibility = 0.06;

    // Parameters
    const params = {
      A1: parseFloat(document.getElementById('A1').value),
      f1: parseFloat(document.getElementById('f1').value),
      p1: parseFloat(document.getElementById('p1').value) * Math.PI/180,
      A2: parseFloat(document.getElementById('A2').value),
      f2: parseFloat(document.getElementById('f2').value),
      p2: parseFloat(document.getElementById('p2').value) * Math.PI/180,
      speed: 1.2,
      animate: true
    };

    // UI value update helpers
    function updateUIValues(){
      document.getElementById('A1val').innerText = params.A1.toFixed(2);
      document.getElementById('f1val').innerText = params.f1.toFixed(2);
      document.getElementById('p1val').innerText = Math.round(params.p1 * 180/Math.PI);
      document.getElementById('A2val').innerText = params.A2.toFixed(2);
      document.getElementById('f2val').innerText = params.f2.toFixed(2);
      document.getElementById('p2val').innerText = Math.round(params.p2 * 180/Math.PI);
    }

    // Create line meshes for waves
    const resolution = 300;
    const xMin = -9, xMax = 9;

    function createLine(name, color) {
      const points = [];
      for (let i=0;i<=resolution;i++) points.push(new BABYLON.Vector3(0,0,0));
      const lines = BABYLON.MeshBuilder.CreateLines(name, {points: points, updatable:true}, scene);
      lines.color = color;
      lines.alwaysSelectAsActiveMesh = true;
      lines.enableEdgesRendering(); // subtle highlight
      return lines;
    }

    const wave1 = createLine("wave1", new BABYLON.Color3(0.24,0.56,0.95));
    const wave2 = createLine("wave2", new BABYLON.Color3(1.0,0.62,0.2));
    const result = createLine("result", new BABYLON.Color3(1,1,1));

    // compute y for a wave at x and time t
    function waveY(A, f, phase, x, t) {
      // simple sinusoidal traveling wave: y = A * sin(2π f (x - vt) + phase)
      // choose v proportional to f for visible patterns
      const v = params.speed;
      return A * Math.sin(2 * Math.PI * f * (x - v * t) + phase);
    }

    // update lines
    function updateLines(t) {
      const pts1 = [];
      const pts2 = [];
      const ptsR = [];
      for (let i=0;i<=resolution;i++){
        const x = xMin + (xMax - xMin) * (i / resolution);
        const y1 = waveY(params.A1, params.f1, params.p1, x, t);
        const y2 = waveY(params.A2, params.f2, params.p2, x, t);
        const yr = y1 + y2;
        // position them a bit above center plane for visibility
        pts1.push(new BABYLON.Vector3(x, y1 - 0.5, 0));
        pts2.push(new BABYLON.Vector3(x, y2 - 1.2, 0)); // offset second for clarity
        ptsR.push(new BABYLON.Vector3(x, yr + 1.2, 0)); // show resultant separated
      }
      BABYLON.MeshBuilder.CreateLines(null, {points: pts1, instance: wave1});
      BABYLON.MeshBuilder.CreateLines(null, {points: pts2, instance: wave2});
      BABYLON.MeshBuilder.CreateLines(null, {points: ptsR, instance: result});
    }

    // initial draw
    updateLines(0);

    // Inputs event listeners
    const map = {
      A1: document.getElementById('A1'),
      f1: document.getElementById('f1'),
      p1: document.getElementById('p1'),
      A2: document.getElementById('A2'),
      f2: document.getElementById('f2'),
      p2: document.getElementById('p2'),
    };
    Object.keys(map).forEach(k=>{
      map[k].addEventListener('input', (e)=>{
        const val = parseFloat(e.target.value);
        if (k.startsWith('p')) params[k] = val * Math.PI/180;
        else params[k] = val;
        updateUIValues();
      });
    });

    // AI Analyze (simple heuristic suggestions)
    document.getElementById('aiAnalyze').addEventListener('click', () => {
      // heuristic: if user wants constructive interference -> match phases and frequencies
      // We'll check current phases: if nearly opposit (180deg) suggest constructive by aligning phase
      const phaseDiff = Math.abs((params.p1 - params.p2 + Math.PI) % (2*Math.PI) - Math.PI);
      let message = "";
      if (phaseDiff > Math.PI*0.75) {
        // almost opposite -> destructive now
        // Suggest parameters for constructive
        params.p2 = params.p1; // align phases
        params.f2 = params.f1;
        message = "AI: Deteksi saat ini cenderung menghasilkan interferensi destruktif. Saran: samakan fase dan frekuensi untuk memperkuat interferensi (kini saya atur p2=p1, f2=f1).";
      } else {
        // already constructive - maybe suggest to explore destruktif
        params.p2 = (params.p1 + Math.PI) % (2*Math.PI);
        message = "AI: Saat ini cenderung konstruktif. Saran eksperimen: ubah fase gelombang kedua sebesar 180° untuk mencoba interferensi destruktif (saya atur p2 = p1 + 180°).";
      }
      updateUIValues();
      // flash a temporary on-screen hint using alert-like panel
      showTemporaryMessage(message, 4200);
    });

    // Toggle animation
    document.getElementById('toggleAnimate').addEventListener('click', ()=>{
      params.animate = !params.animate;
      showTemporaryMessage("Animasi " + (params.animate ? "dihidupkan" : "dimatikan"), 1600);
    });

    // Temporary message display
    let msgDiv = null;
    function showTemporaryMessage(txt, time=2000) {
      if (!msgDiv) {
        msgDiv = document.createElement('div');
        msgDiv.style.position = 'absolute';
        msgDiv.style.right = '12px';
        msgDiv.style.top = '12px';
        msgDiv.style.zIndex = '30';
        msgDiv.style.background = 'rgba(2,6,23,0.85)';
        msgDiv.style.padding = '10px 12px';
        msgDiv.style.borderRadius = '8px';
        msgDiv.style.color = '#e6eef8';
        msgDiv.style.maxWidth = '360px';
        msgDiv.style.fontSize = '13px';
        document.body.appendChild(msgDiv);
      }
      msgDiv.innerText = txt;
      msgDiv.style.opacity = '1';
      setTimeout(()=>{ msgDiv.style.transition = 'opacity 0.6s'; msgDiv.style.opacity='0'; }, time);
    }

    // Animation loop variables
    let start = performance.now() / 1000;
    engine.runRenderLoop(function(){
      const now = performance.now() / 1000;
      const t = (now - start) * 1.0;
      if (params.animate) updateLines(t);
      scene.render();
    });

    // Resize
    window.addEventListener('resize', function(){ engine.resize(); });

    // Helpful initial UI values
    updateUIValues();

    // Accessibility: keyboard shortcuts
    window.addEventListener('keydown', (ev)=>{
      if (ev.key === ' ') { params.animate = !params.animate; showTemporaryMessage("Toggle animasi: " + params.animate, 1200); ev.preventDefault(); }
    });

    // Optional: WebXR quick init (only if device supports)
    async function tryInitXR() {
      if (BABYLON.WebXRDefaultExperience) {
        try {
          const xr = await scene.createDefaultXRExperienceAsync({floorMeshes: [grid]});
          console.log("WebXR siap.", xr);
          // simple UI hint
          showTemporaryMessage("WebXR: tombol enter VR tersedia di kanan atas bila perangkat mendukung", 4000);
        } catch (e) {
          // ignore if unsupported
          console.log("WebXR tidak tersedia/diizinkan:", e);
        }
      }
    }
    tryInitXR();

    // End IIFE
  })();
  </script>
</body>
</html>
