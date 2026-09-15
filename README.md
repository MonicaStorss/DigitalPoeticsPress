# WebAR Text Trigger — Prototype
**Digital Poetics Press · by Monica Storss**

A single-file WebAR app. Point a phone camera at printed text (or any image-based marker); overlay a JPG/PNG or .glb 3D model on top of it in augmented reality. No app store. No server. Runs in the browser.

---

## How it works

This prototype uses **MindAR.js** (image tracking) + **A-Frame** (WebXR scene). Everything — including the target compilation step — runs locally in the browser. Nothing is uploaded anywhere.

**The "text as trigger" concept:**
MindAR does *image* tracking. The trigger doesn't have to be a QR code — it can be a photo of any high-contrast printed surface, including a page of poetry, a zine spread, a chapbook cover, or a handwritten note. The more visual contrast and unique detail, the more reliable the tracking.

---

## To test locally

You need HTTPS (camera access requires it). The fastest way:

```bash
# If you have Node.js:
npx serve .
# then open https://localhost:3000

# Or Python:
python3 -m http.server 8080
# then open https://localhost:8080
```

Or drop `index.html` into:
- **Netlify Drop** (drag-and-drop deploy → netlify.com/drop) — gives you an instant HTTPS URL
- **GitHub Pages** (push to a repo, enable Pages)
- **Vercel** (drag and drop)

---

## Workflow

1. **Print your trigger text** — a poem, a page, a postcard
2. **Photograph it** with good light, flat-on (or use an existing high-res scan)
3. Open the app → upload the photo as your trigger image
4. Watch it compile (extracts visual features, runs in your browser — 5–20 seconds)
5. Upload your overlay: a `.jpg/.png` image *or* a `.glb` 3D model
6. Adjust scale and position with the sliders
7. Hit **Launch AR** — point your phone camera at the printed original

---

## Tips for better tracking

| Do | Avoid |
|---|---|
| High-contrast printed text | Glossy reflective surfaces |
| Lots of unique visual detail (varied letterforms help) | Repetitive patterns, large solid areas |
| Flat, well-lit surface | Crumpled or curved paper |
| At least A5 / half-letter size | Tiny thumbnails |

---

## Overlay types

**Image (JPG/PNG):** Renders as a flat plane anchored to the trigger. Good for: additional text layers, illustrations, photographic content, animated GIFs (partial A-Frame support).

**3D Model (GLB):** Renders a full 3D object anchored to the trigger. Good for: sculptural objects, animated characters, spatial environments. Free GLB sources: [Sketchfab](https://sketchfab.com/features/free-3d-models), [Google Poly archive](https://poly.pizza), [Khronos sample models](https://github.com/KhronosGroup/glTF-Sample-Models).

---

## Extending this prototype

- **Multiple triggers / overlays:** MindAR supports multiple `targetIndex` values — compile multiple images at once and map a different overlay to each
- **Video overlay:** Replace `<a-image>` with `<a-video>` and a `<video>` asset
- **Text overlay:** Use `<a-text>` in the A-Frame entity for AR poetry rendered in type
- **Animation:** Add `animation` component to the overlay entity for hover/spin effects
- **Sound:** Add `<audio>` asset + `sound` component to trigger audio on target find

---

## Stack

- [MindAR.js 1.2.5](https://hiukim.github.io/mind-ar-js-doc/) — image tracking, in-browser compilation
- [A-Frame 1.4.2](https://aframe.io) — WebXR scene graph
- No build step. No dependencies to install. One HTML file.

---

*All processing local. No data leaves the device.*
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8" />
  <meta name="viewport" content="width=device-width, initial-scale=1.0, user-scalable=no" />
  <title>Digital Poetics | WebAR Text Trigger</title>

  <!-- MindAR image tracking + A-Frame -->
  <script src="https://cdn.jsdelivr.net/npm/mind-ar@1.2.5/dist/mindar-image.prod.js"></script>
  <script src="https://aframe.io/releases/1.4.2/aframe.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/mind-ar@1.2.5/dist/mindar-image-aframe.prod.js"></script>

  <style>
    *, *::before, *::after { box-sizing: border-box; margin: 0; padding: 0; }

    :root {
      --ink:    #0d0d0d;
      --paper:  #f7f4ee;
      --ghost:  #e8e2d5;
      --accent: #c0392b;
      --dim:    #8a8070;
      --radius: 4px;
      --mono:   'Courier New', Courier, monospace;
      --sans:   'Georgia', serif;
    }

    html, body {
      height: 100%;
      background: var(--ink);
      color: var(--paper);
      font-family: var(--sans);
      overflow: hidden;
    }

    /* ── SETUP SCREEN ──────────────────────────────── */
    #setup {
      position: fixed;
      inset: 0;
      z-index: 100;
      background: var(--ink);
      display: flex;
      flex-direction: column;
      align-items: center;
      justify-content: flex-start;
      overflow-y: auto;
      padding: 2rem 1.5rem 3rem;
      gap: 0;
    }

    .wordmark {
      font-family: var(--mono);
      font-size: 0.72rem;
      letter-spacing: 0.25em;
      text-transform: uppercase;
      color: var(--dim);
      margin-bottom: 2.5rem;
      text-align: center;
    }

    h1 {
      font-size: clamp(1.6rem, 5vw, 2.4rem);
      font-weight: 400;
      line-height: 1.15;
      text-align: center;
      color: var(--paper);
      margin-bottom: 0.5rem;
    }

    .subhead {
      font-family: var(--mono);
      font-size: 0.8rem;
      color: var(--dim);
      text-align: center;
      margin-bottom: 2.5rem;
      line-height: 1.6;
      max-width: 34ch;
    }

    /* steps */
    .steps {
      width: 100%;
      max-width: 480px;
      display: flex;
      flex-direction: column;
      gap: 1.25rem;
    }

    .step {
      background: #181818;
      border: 1px solid #2a2a2a;
      border-radius: var(--radius);
      padding: 1.25rem 1.4rem;
      position: relative;
      transition: border-color 0.2s;
    }
    .step.done   { border-color: #2d5a27; }
    .step.active { border-color: #4a4a4a; }

    .step-label {
      font-family: var(--mono);
      font-size: 0.65rem;
      letter-spacing: 0.2em;
      text-transform: uppercase;
      color: var(--dim);
      margin-bottom: 0.5rem;
    }

    .step h2 {
      font-size: 1rem;
      font-weight: 400;
      color: var(--paper);
      margin-bottom: 0.35rem;
    }

    .step p {
      font-family: var(--mono);
      font-size: 0.75rem;
      color: var(--dim);
      line-height: 1.55;
      margin-bottom: 0.9rem;
    }

    .upload-zone {
      border: 1px dashed #3a3a3a;
      border-radius: var(--radius);
      padding: 1.1rem;
      text-align: center;
      cursor: pointer;
      transition: border-color 0.2s, background 0.2s;
      font-family: var(--mono);
      font-size: 0.78rem;
      color: var(--dim);
    }
    .upload-zone:hover { border-color: var(--accent); color: var(--paper); background: #1f1f1f; }
    .upload-zone.loaded { border-color: #2d5a27; color: #6abf69; }

    input[type="file"] { display: none; }

    .preview-thumb {
      width: 60px;
      height: 40px;
      object-fit: cover;
      border-radius: 2px;
      display: none;
      margin: 0.5rem auto 0;
    }
    .preview-thumb.visible { display: block; }

    .type-toggle {
      display: flex;
      gap: 0.5rem;
      margin-bottom: 0.9rem;
    }
    .type-btn {
      flex: 1;
      padding: 0.55rem;
      border: 1px solid #3a3a3a;
      background: transparent;
      color: var(--dim);
      font-family: var(--mono);
      font-size: 0.75rem;
      border-radius: var(--radius);
      cursor: pointer;
      transition: all 0.15s;
    }
    .type-btn.active {
      border-color: var(--accent);
      color: var(--paper);
      background: #1a0f0f;
    }

    /* compile progress */
    #compile-bar-wrap {
      display: none;
      margin-top: 0.75rem;
    }
    #compile-bar-wrap.visible { display: block; }
    .bar-track {
      background: #2a2a2a;
      border-radius: 2px;
      height: 3px;
      overflow: hidden;
      margin-bottom: 0.4rem;
    }
    .bar-fill {
      height: 100%;
      background: var(--accent);
      width: 0%;
      transition: width 0.3s ease;
    }
    .bar-label {
      font-family: var(--mono);
      font-size: 0.68rem;
      color: var(--dim);
    }

    /* launch button */
    #launch-btn {
      margin-top: 1.5rem;
      width: 100%;
      max-width: 480px;
      padding: 1rem;
      background: var(--accent);
      color: #fff;
      border: none;
      border-radius: var(--radius);
      font-family: var(--mono);
      font-size: 0.85rem;
      letter-spacing: 0.1em;
      text-transform: uppercase;
      cursor: pointer;
      opacity: 0.35;
      pointer-events: none;
      transition: opacity 0.2s;
    }
    #launch-btn.ready {
      opacity: 1;
      pointer-events: auto;
    }
    #launch-btn.ready:hover { background: #a93226; }

    .footer-note {
      margin-top: 1.5rem;
      font-family: var(--mono);
      font-size: 0.65rem;
      color: #444;
      text-align: center;
      max-width: 36ch;
      line-height: 1.6;
    }

    /* ── AR OVERLAY ────────────────────────────────── */
    #ar-container {
      position: fixed;
      inset: 0;
      z-index: 200;
      display: none;
    }
    #ar-container.visible { display: block; }

    /* back button */
    #back-btn {
      position: fixed;
      top: 1rem;
      left: 1rem;
      z-index: 300;
      background: rgba(0,0,0,0.65);
      color: var(--paper);
      border: 1px solid #3a3a3a;
      border-radius: var(--radius);
      padding: 0.5rem 0.85rem;
      font-family: var(--mono);
      font-size: 0.72rem;
      letter-spacing: 0.1em;
      cursor: pointer;
      backdrop-filter: blur(6px);
      display: none;
    }
    #back-btn.visible { display: block; }

    /* scanning hint */
    #scan-hint {
      position: fixed;
      bottom: 2rem;
      left: 50%;
      transform: translateX(-50%);
      z-index: 300;
      background: rgba(0,0,0,0.7);
      color: var(--paper);
      border: 1px solid #3a3a3a;
      border-radius: 999px;
      padding: 0.55rem 1.25rem;
      font-family: var(--mono);
      font-size: 0.75rem;
      letter-spacing: 0.08em;
      backdrop-filter: blur(6px);
      display: none;
      white-space: nowrap;
      animation: pulse 2s ease-in-out infinite;
    }
    #scan-hint.visible { display: block; }
    @keyframes pulse {
      0%, 100% { opacity: 0.75; }
      50%       { opacity: 1; }
    }

    a-scene { height: 100vh !important; }
  </style>
</head>
<body>

<!-- ══════════════════════════════════════════════
     SETUP SCREEN
═══════════════════════════════════════════════ -->
<div id="setup">

  <div class="wordmark">Digital Poetics Press · WebAR Prototype</div>

  <h1>Text as Trigger</h1>
  <p class="subhead">Point your camera at a printed page.<br>Place a world on top of words.</p>

  <div class="steps">

    <!-- STEP 1: trigger image -->
    <div class="step active" id="step1">
      <div class="step-label">Step 1 of 3</div>
      <h2>Photograph your trigger text</h2>
      <p>Upload a photo of the physical text you want to use as an AR marker — a printed poem, a book page, a zine spread. Higher contrast = better tracking.</p>
      <label class="upload-zone" id="trigger-zone" for="trigger-file">
        <span id="trigger-zone-label">↑ Choose trigger image (JPG/PNG)</span>
        <img id="trigger-preview" class="preview-thumb" alt="trigger preview" />
      </label>
      <input type="file" id="trigger-file" accept="image/*" />

      <div id="compile-bar-wrap">
        <div class="bar-track"><div class="bar-fill" id="compile-bar"></div></div>
        <div class="bar-label" id="compile-label">Compiling target…</div>
      </div>
    </div>

    <!-- STEP 2: content to overlay -->
    <div class="step" id="step2">
      <div class="step-label">Step 2 of 3</div>
      <h2>Choose your overlay content</h2>
      <p>Upload either a flat image (JPG/PNG) that will float above the trigger, or a 3D model (GLB) that will anchor to it in space.</p>

      <div class="type-toggle">
        <button class="type-btn active" id="btn-image" onclick="setContentType('image')">Image (JPG/PNG)</button>
        <button class="type-btn" id="btn-model" onclick="setContentType('model')">3D Model (GLB)</button>
      </div>

      <label class="upload-zone" id="content-zone" for="content-file">
        <span id="content-zone-label">↑ Choose overlay file</span>
        <img id="content-preview" class="preview-thumb" alt="content preview" />
      </label>
      <input type="file" id="content-file" accept="image/*,.glb" />
    </div>

    <!-- STEP 3: scale / position tweaks -->
    <div class="step" id="step3">
      <div class="step-label">Step 3 of 3</div>
      <h2>Adjust placement</h2>
      <p>Scale controls how large the overlay appears relative to the trigger image. Position offsets shift it in 3D space.</p>

      <div style="display:grid; grid-template-columns:1fr 1fr; gap:0.75rem; font-family:var(--mono); font-size:0.78rem; color:var(--dim);">
        <div>
          <label for="s-scale">Scale</label><br/>
          <input type="range" id="s-scale" min="0.1" max="3" step="0.05" value="1"
            style="width:100%; accent-color:var(--accent); margin-top:0.3rem;"
            oninput="document.getElementById('v-scale').textContent=this.value" />
          <span id="v-scale">1</span>
        </div>
        <div>
          <label for="s-y">Height offset (Y)</label><br/>
          <input type="range" id="s-y" min="-1" max="1" step="0.05" value="0"
            style="width:100%; accent-color:var(--accent); margin-top:0.3rem;"
            oninput="document.getElementById('v-y').textContent=this.value" />
          <span id="v-y">0</span>
        </div>
        <div>
          <label for="s-rx">Tilt (X rotation°)</label><br/>
          <input type="range" id="s-rx" min="-180" max="180" step="5" value="0"
            style="width:100%; accent-color:var(--accent); margin-top:0.3rem;"
            oninput="document.getElementById('v-rx').textContent=this.value" />
          <span id="v-rx">0</span>
        </div>
        <div>
          <label for="s-ry">Spin (Y rotation°)</label><br/>
          <input type="range" id="s-ry" min="-180" max="180" step="5" value="0"
            style="width:100%; accent-color:var(--accent); margin-top:0.3rem;"
            oninput="document.getElementById('v-ry').textContent=this.value" />
          <span id="v-ry">0</span>
        </div>
      </div>
    </div>

  </div><!-- /steps -->

  <button id="launch-btn" onclick="launchAR()">▶ Launch AR Experience</button>

  <p class="footer-note">All processing runs locally in your browser.<br>Nothing is uploaded to any server.</p>

</div><!-- /setup -->

<!-- ══════════════════════════════════════════════
     AR CONTAINER (injected dynamically)
═══════════════════════════════════════════════ -->
<div id="ar-container"></div>
<button id="back-btn" onclick="stopAR()">← Back</button>
<div id="scan-hint">Point camera at your trigger text</div>

<script>
  // ── STATE ──────────────────────────────────────
  let compiledMindData = null;   // ArrayBuffer from MindAR compiler
  let contentType = 'image';     // 'image' | 'model'
  let contentObjectURL = null;   // blob URL for the overlay asset
  let triggerObjectURL = null;
  let arScene = null;

  // ── CONTENT TYPE TOGGLE ───────────────────────
  function setContentType(t) {
    contentType = t;
    document.getElementById('btn-image').classList.toggle('active', t === 'image');
    document.getElementById('btn-model').classList.toggle('active', t === 'model');
    // reset content file
    document.getElementById('content-file').accept = t === 'image' ? 'image/*' : '.glb';
    document.getElementById('content-file').value = '';
    document.getElementById('content-zone').classList.remove('loaded');
    document.getElementById('content-zone-label').textContent = '↑ Choose overlay file';
    document.getElementById('content-preview').classList.remove('visible');
    contentObjectURL = null;
    checkReady();
  }

  // ── TRIGGER FILE ──────────────────────────────
  document.getElementById('trigger-file').addEventListener('change', async function(e) {
    const file = e.target.files[0];
    if (!file) return;

    // show thumb
    triggerObjectURL = URL.createObjectURL(file);
    const prev = document.getElementById('trigger-preview');
    prev.src = triggerObjectURL;
    prev.classList.add('visible');

    document.getElementById('trigger-zone').classList.remove('loaded');
    document.getElementById('trigger-zone-label').textContent = 'Compiling target…';

    // show progress bar
    const barWrap  = document.getElementById('compile-bar-wrap');
    const bar      = document.getElementById('compile-bar');
    const barLabel = document.getElementById('compile-label');
    barWrap.classList.add('visible');
    bar.style.width = '5%';
    barLabel.textContent = 'Loading image…';

    try {
      // MindAR compiler lives on window after script loads
      const { MindARThree } = window.MINDAR?.IMAGE ?? {};
      // Compiler is in mindar-image.prod.js as window.MINDAR.IMAGE.Compiler
      const Compiler = window.MINDAR?.IMAGE?.Compiler;
      if (!Compiler) throw new Error('MindAR compiler not available');

      const compiler = new Compiler();

      // Load image
      bar.style.width = '20%';
      barLabel.textContent = 'Parsing image…';
      const img = await loadImage(triggerObjectURL);

      bar.style.width = '35%';
      barLabel.textContent = 'Extracting features…';
      await compiler.compileImageTargets([img], (progress) => {
        bar.style.width = (35 + progress * 0.55) + '%';
        barLabel.textContent = `Compiling… ${Math.round(progress)}%`;
      });

      bar.style.width = '95%';
      barLabel.textContent = 'Exporting target data…';
      compiledMindData = await compiler.exportData();

      bar.style.width = '100%';
      barLabel.textContent = 'Target ready ✓';
      document.getElementById('trigger-zone').classList.add('loaded');
      document.getElementById('trigger-zone-label').textContent = '✓ ' + file.name;
      document.getElementById('step1').classList.add('done');
      document.getElementById('step2').classList.add('active');

    } catch(err) {
      bar.style.width = '0%';
      barLabel.textContent = 'Error: ' + err.message;
      console.error(err);
    }
    checkReady();
  });

  function loadImage(src) {
    return new Promise((res, rej) => {
      const img = new Image();
      img.onload = () => res(img);
      img.onerror = rej;
      img.src = src;
    });
  }

  // ── CONTENT FILE ─────────────────────────────
  document.getElementById('content-file').addEventListener('change', function(e) {
    const file = e.target.files[0];
    if (!file) return;
    if (contentObjectURL) URL.revokeObjectURL(contentObjectURL);
    contentObjectURL = URL.createObjectURL(file);
    document.getElementById('content-zone').classList.add('loaded');
    document.getElementById('content-zone-label').textContent = '✓ ' + file.name;
    document.getElementById('step2').classList.add('done');
    document.getElementById('step3').classList.add('active');

    // show image preview for images
    if (contentType === 'image') {
      const prev = document.getElementById('content-preview');
      prev.src = contentObjectURL;
      prev.classList.add('visible');
    }
    checkReady();
  });

  // ── READY CHECK ───────────────────────────────
  function checkReady() {
    const ready = compiledMindData && contentObjectURL;
    document.getElementById('launch-btn').classList.toggle('ready', !!ready);
  }

  // ── LAUNCH AR ─────────────────────────────────
  async function launchAR() {
    if (!compiledMindData || !contentObjectURL) return;

    // Read placement values
    const scale  = parseFloat(document.getElementById('s-scale').value);
    const yOff   = parseFloat(document.getElementById('s-y').value);
    const rx     = parseFloat(document.getElementById('s-rx').value);
    const ry     = parseFloat(document.getElementById('s-ry').value);

    // Convert mind data → blob URL
    const mindBlob = new Blob([compiledMindData]);
    const mindURL  = URL.createObjectURL(mindBlob);

    // Build A-Frame scene HTML
    const overlayEl = buildOverlayElement(scale, yOff, rx, ry);

    const sceneHTML = `
      <a-scene
        mindar-image="imageTargetSrc: ${mindURL}; autoStart: true; uiLoading: no; uiScanning: no; uiError: no;"
        color-space="sRGB"
        renderer="colorManagement: true, physicallyCorrectLights: true"
        vr-mode-ui="enabled: false"
        device-orientation-permission-ui="enabled: false"
      >
        <a-assets>
          ${contentType === 'image'
            ? `<img id="overlay-img" src="${contentObjectURL}" crossorigin="anonymous" />`
            : `<a-asset-item id="overlay-model" src="${contentObjectURL}"></a-asset-item>`
          }
        </a-assets>

        <a-camera position="0 0 0" look-controls="enabled: false"></a-camera>

        <a-entity mindar-image-target="targetIndex: 0">
          ${overlayEl}
        </a-entity>
      </a-scene>
    `;

    const container = document.getElementById('ar-container');
    container.innerHTML = sceneHTML;
    container.classList.add('visible');
    document.getElementById('setup').style.display = 'none';
    document.getElementById('back-btn').classList.add('visible');
    document.getElementById('scan-hint').classList.add('visible');

    // Fade hint after 6s once target is found
    const scene = container.querySelector('a-scene');
    scene.addEventListener('targetFound', () => {
      document.getElementById('scan-hint').style.opacity = '0';
      setTimeout(() => document.getElementById('scan-hint').classList.remove('visible'), 600);
    });
    scene.addEventListener('targetLost', () => {
      document.getElementById('scan-hint').classList.add('visible');
      document.getElementById('scan-hint').style.opacity = '';
    });
  }

  function buildOverlayElement(scale, yOff, rx, ry) {
    const pos = `0 ${yOff} 0`;
    const rot = `${rx} ${ry} 0`;
    if (contentType === 'image') {
      return `<a-image src="#overlay-img"
                  position="${pos}"
                  rotation="${rot}"
                  scale="${scale} ${scale} ${scale}"
                  opacity="0.95">
              </a-image>`;
    } else {
      return `<a-gltf-model src="#overlay-model"
                  position="${pos}"
                  rotation="${rot}"
                  scale="${scale} ${scale} ${scale}">
              </a-gltf-model>`;
    }
  }

  // ── STOP AR ───────────────────────────────────
  function stopAR() {
    const container = document.getElementById('ar-container');

    // Destroy A-Frame scene to release camera
    const scene = container.querySelector('a-scene');
    if (scene) {
      try { scene.systems['mindar-image-system']?.stop?.(); } catch(e) {}
      scene.parentNode.removeChild(scene);
    }
    container.innerHTML = '';
    container.classList.remove('visible');
    document.getElementById('setup').style.display = '';
    document.getElementById('back-btn').classList.remove('visible');
    document.getElementById('scan-hint').classList.remove('visible');
    document.getElementById('scan-hint').style.opacity = '';
  }
</script>
</body>
</html>
