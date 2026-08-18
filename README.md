# GageMikey.github.io
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Basic Background Generator for Josh</title>
  <script src="https://cdnjs.cloudflare.com/ajax/libs/jszip/3.10.1/jszip.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/mp4-muxer@5.1.0/build/mp4-muxer.min.js"></script>
  <script src="https://cdn.jsdelivr.net/npm/webm-muxer@3.1.0/build/webm-muxer.min.js"></script>
  <style>
    body { background: #fff; color: #000; display: flex; gap: 20px; margin: 0; padding: 20px; font-family: sans-serif; }
    #pfp-container { width: 512px; height: 512px; border: 1px solid #000; background: #ccc; position: relative; flex-shrink: 0; }
    canvas { max-width: 100%; max-height: 100%; display: block; }
    #canvas-overlay { position: absolute; top: 0; left: 0; width: 100%; height: 100%; pointer-events: auto; }
    .handle-line { stroke: #000; stroke-width: 1; stroke-dasharray: 2 2; }
    .handle-circle { fill: #fff; stroke: #000; stroke-width: 1; cursor: grab; }
    .handle-circle:hover, .handle-circle.active { fill: #ff0; }
    .controls { display: flex; flex-direction: column; gap: 10px; background: #f0f0f0; padding: 15px; border: 1px solid #000; width: 500px; }
    .slider-row { display: flex; justify-content: space-between; align-items: center; font-size: 14px; }
    .slider-row label { width: 180px; }
    input[type="range"] { flex-grow: 1; }
    input[type="number"], input[type="text"], select { padding: 2px; }
    .val-display { min-width: 40px; text-align: right; }
    .gradient-editor-box { border: 1px solid #000; padding: 10px; display: flex; flex-direction: column; gap: 10px; background: #e8e8e8; }
    .gradient-ramp-container { position: relative; width: 100%; height: 40px; }
    .gradient-ramp-track { width: 100%; height: 20px; border: 1px solid #000; cursor: crosshair; position: relative; }
    .gradient-stop-handle { position: absolute; top: 22px; transform: translateX(-50%); width: 12px; height: 12px; cursor: grab; z-index: 10; }
    .gradient-stop-handle.active { z-index: 20; }
    .stop-pin { width: 12px; height: 12px; border: 1px solid #000; background: #fff; }
    .gradient-stop-handle.active .stop-pin { border: 2px solid #f00; }
    button, .btn-file { padding: 5px; background: #ddd; border: 1px solid #000; color: #000; cursor: pointer; text-align: center; text-decoration: none; font-size: 14px; }
    button:hover, .btn-file:hover { background: #ccc; }
    button:disabled { color: #888; cursor: default; }
    input[type="file"] { display: none; }
    .btn-group { display: flex; gap: 10px; margin-top: 10px; border-top: 1px solid #000; padding-top: 10px; }
    .action-btn { flex: 1; padding: 8px; font-weight: bold; }
    .custom-res-group { display: none; gap: 5px; }
    .gradient-only-row { display: flex; }
    .progress-container { display: none; flex-direction: column; gap: 5px; margin-top: 10px; }
    .progress-bar-bg { width: 100%; height: 15px; border: 1px solid #000; background: #fff; }
    .progress-bar-fill { height: 100%; width: 0%; background: #00f; }
    .progress-text { font-size: 12px; }
    .section-header { font-weight: bold; border-bottom: 1px solid #000; margin-top: 10px; padding-bottom: 2px; }
  </style>
</head>
<body>

<div id="pfp-container">
  <canvas id="glcanvas" width="512" height="512"></canvas>
  <svg id="canvas-overlay" viewBox="0 0 512 512" preserveAspectRatio="none"></svg>
</div>

<div class="controls">
  <div class="section-header">Canvas & Layout</div>

  <div class="slider-row">
    <label for="res-preset">Resolution Preset:</label>
    <select id="res-preset" onchange="handleResolutionChange(this.value)">
      <option value="512x512" selected>512x512 (Standard PFP)</option>
      <option value="1080x1080">1080x1080 (HD Square)</option>
      <option value="1920x1080">1920x1080 (16:9 Landscape Video)</option>
      <option value="1080x1920">1080x1920 (9:16 Shorts/Reels/TikTok)</option>
      <option value="1080x1350">1080x1350 (4:5 Portrait Post)</option>
      <option value="custom">Custom...</option>
    </select>
  </div>

  <div class="slider-row custom-res-group" id="custom-res-controls">
    <label>Custom Size W x H:</label>
    <input type="number" id="custom-width" value="1080" min="64" max="4096">
    <span>x</span>
    <input type="number" id="custom-height" value="1080" min="64" max="4096">
    <button onclick="applyCustomResolution()">Apply</button>
  </div>

  <div class="slider-row">
    <label for="toggle-handles">Show On-Canvas Handles:</label>
    <input type="checkbox" id="toggle-handles" checked onchange="toggleHandlesVisibility(this.checked)">
  </div>

  <div class="section-header">Background Controls</div>

  <div class="slider-row">
    <label for="bg-type">Fill Type:</label>
    <select id="bg-type" onchange="updateBgType(this.value)">
      <option value="solid">Solid Color</option>
      <option value="linear" selected>Linear Gradient</option>
    </select>
  </div>

  <div class="slider-row" id="row-solid-color" style="display: none;">
    <label for="solid-color">Background Color:</label>
    <input type="color" id="solid-color" value="#1e293b" oninput="updateSolidColor(this.value)">
    <span id="val-solid-hex" class="val-display">#1E293B</span>
  </div>

  <div class="gradient-editor-box" id="gradient-controls-wrapper">
    <div style="display: flex; justify-content: space-between; align-items: center;">
      <span>GRADIENT RAMP (Click to add)</span>
      <span id="stop-count-badge">2 Stops</span>
    </div>

    <div class="gradient-ramp-container">
      <div class="gradient-ramp-track" id="gradient-ramp-track" onclick="handleTrackClick(event)"></div>
    </div>

    <div class="slider-row" style="border: 1px solid #000; padding: 5px;">
      <label style="width: auto;">Stop:</label>
      <input type="color" id="active-stop-color" value="#3b82f6" oninput="updateActiveStopColor(this.value)">
      <input type="text" id="active-stop-hex" value="#3B82F6" style="width: 70px;" onchange="updateActiveStopHex(this.value)">
      <label style="width: auto; margin-left: 5px;">Pos:</label>
      <input type="range" id="active-stop-pos" min="0" max="100" value="0" oninput="updateActiveStopPos(this.value)" style="width: 80px;">
      <span id="val-stop-pos" class="val-display" style="min-width: 35px;">0%</span>
      <button type="button" id="btn-delete-stop" onclick="deleteActiveStop()" style="margin-left: 5px;">Delete</button>
    </div>

    <div style="display: flex; gap: 5px;">
      <button type="button" style="flex: 1;" onclick="addGradientStopAtCenter()">Add Stop</button>
      <button type="button" style="flex: 1;" onclick="reverseGradientStops()">Reverse</button>
      <button type="button" style="flex: 1;" onclick="distributeGradientStops()">Space Evenly</button>
    </div>

    <div class="slider-row">
      <label for="bg-easing">Gradient Easing Curve:</label>
      <select id="bg-easing" onchange="updateGradientEasing(this.value)">
        <option value="0" selected>Linear Interpolation</option>
        <option value="1">Smoothstep</option>
        <option value="2">Power Curve</option>
      </select>
    </div>

    <div class="slider-row gradient-only-row" id="row-bg-angle">
      <label for="bg-angle">Angle:</label>
      <input type="range" id="bg-angle" min="0" max="360" step="1" value="45" oninput="updateBgAngle(this.value)">
      <span id="val-bg-angle" class="val-display">45°</span>
    </div>

    <div class="slider-row gradient-only-row" id="row-bg-spread">
      <label for="bg-spread">Blend Sharpness:</label>
      <input type="range" id="bg-spread" min="0.1" max="3.0" step="0.05" value="1.0" oninput="updateBgSpread(this.value)">
      <span id="val-bg-spread" class="val-display">1.00</span>
    </div>
  </div>
</div>

<div class="controls">
  <div class="section-header">Pattern Controls</div>

  <div style="display: flex; flex-direction: column; gap: 5px;">
    <label class="btn-file" for="pattern-file-input" id="pattern-upload-label">Upload Pattern Icon</label>
    <input type="file" id="pattern-file-input" accept="image/png, image/jpeg, image/webp" onchange="handlePatternUpload(event)">
  </div>

  <div class="slider-row">
    <label for="pattern-dir">Direction:</label>
    <select id="pattern-dir" onchange="updatePatternDirection(this.value)">
      <option value="right" selected>Right</option>
      <option value="up-right">Up-Right</option>
      <option value="up">Up</option>
      <option value="up-left">Up-Left</option>
      <option value="left">Left</option>
      <option value="down-left">Down-Left</option>
      <option value="down">Down</option>
      <option value="down-right">Down-Right</option>
    </select>
  </div>

  <div class="slider-row">
    <label for="pattern-opacity">Opacity:</label>
    <input type="range" id="pattern-opacity" min="0.0" max="0.8" step="0.02" value="0.22" oninput="updateUniform('uPatternOpacity', this.value, 'val-po', '')">
    <span id="val-po" class="val-display">0.22</span>
  </div>

  <div class="slider-row">
    <label for="pattern-scale">Spacing/Count:</label>
    <input type="range" id="pattern-scale" min="2.0" max="12.0" step="0.5" value="3.5" oninput="updateUniform('uPatternScale', this.value, 'val-ps', '')">
    <span id="val-ps" class="val-display">3.50</span>
  </div>

  <div class="slider-row">
    <label for="pattern-speed">Loop Cycles:</label>
    <input type="range" id="pattern-speed" min="1" max="10" step="1" value="2" oninput="updateUniform('uPatternSpeed', this.value, 'val-pms', 'x')">
    <span id="val-pms" class="val-display">2x</span>
  </div>

  <div class="section-header">Export</div>

  <div class="slider-row">
    <label for="export-filename">File Name Prefix:</label>
    <input type="text" id="export-filename" value="BasicBG" style="width: 150px;" placeholder="BasicBG">
  </div>

  <div class="slider-row">
    <label for="export-duration">Duration:</label>
    <select id="export-duration" onchange="updateExportSettings()">
      <option value="4" selected>4 Seconds</option>
      <option value="8">8 Seconds</option>
    </select>
  </div>

  <div class="slider-row">
    <label for="export-fps">Framerate:</label>
    <select id="export-fps" onchange="updateExportSettings()">
      <option value="15">15 FPS</option>
      <option value="24">24 FPS</option>
      <option value="30" selected>30 FPS</option>
      <option value="60">60 FPS</option>
    </select>
  </div>

  <div class="slider-row">
    <label for="export-format">Format:</label>
    <select id="export-format">
      <option value="mp4" selected>MP4 Video</option>
      <option value="webm">WebM Video</option>
      <option value="zip">PNG Sequence</option>
    </select>
  </div>

  <div class="progress-container" id="progress-container">
    <div class="progress-bar-bg">
      <div class="progress-bar-fill" id="progress-fill"></div>
    </div>
    <div class="progress-text" id="progress-text">Preparing...</div>
  </div>

  <div class="btn-group">
    <button class="action-btn" id="pause-btn" onclick="togglePause()">Pause Preview</button>
    <button class="action-btn" id="export-btn" onclick="triggerExport()">Export File</button>
  </div>
</div>

<script>
  const $ = id => document.getElementById(id);
  const canvas = $('glcanvas');
  const overlaySvg = $('canvas-overlay');
  const gl = canvas.getContext('webgl', { preserveDrawingBuffer: true });

  const vsSource = `
    attribute vec2 aPosition;
    varying vec2 vUv;
    void main() {
      vUv = aPosition * 0.5 + 0.5;
      gl_Position = vec4(aPosition, 0.0, 1.0);
    }
  `;

  const fsSource = `
    precision highp float;
    varying vec2 vUv;
    uniform vec2 uResolution;
    uniform float uScreenAspect, uTime, uLoopDuration;
    uniform int uBgType, uStopCount, uEasingMode;
    uniform vec3 uBgColors[8];
    uniform float uBgPositions[8];
    uniform vec2 uGradStart, uGradEnd, uPatternDir;
    uniform float uBgAngle, uBgSpread, uPatternOpacity, uPatternScale, uPatternSpeed;
    uniform sampler2D uPatternTex;
    uniform bool uHasPatternTex;

    vec3 sampleGradient(float t) {
      t = clamp(t, 0.0, 1.0);
      if (uEasingMode == 1) t = smoothstep(0.0, 1.0, t);
      else if (uEasingMode == 2) t = pow(t, max(0.01, uBgSpread));
      if (uStopCount <= 1 || t <= uBgPositions[0]) return uBgColors[0];

      for (int i = 0; i < 7; i++) {
        if (i >= uStopCount - 1) break;
        float p1 = uBgPositions[i], p2 = uBgPositions[i+1];
        if (t >= p1 && t <= p2) {
          return mix(uBgColors[i], uBgColors[i+1], (t - p1) / max(0.00001, (p2 - p1)));
        }
      }
      for (int i = 0; i < 8; i++) {
        if (i == uStopCount - 1) return uBgColors[i];
      }
      return uBgColors[0];
    }

    void main() {
      vec3 col = uBgColors[0];

      if (uBgType == 1) {
        vec2 dir = uGradEnd - uGradStart;
        float lenSq = dot(dir, dir);
        float t = lenSq > 0.00001 ? dot(vUv - uGradStart, dir) / lenSq : dot(vUv - vec2(0.5), vec2(cos(uBgAngle * 0.017453), sin(uBgAngle * 0.017453))) + 0.5;
        col = sampleGradient(t);
      }

      if (uHasPatternTex && uPatternOpacity > 0.0) {
        float progress = fract(uTime / uLoopDuration);
        float cycles = max(1.0, floor(uPatternSpeed + 0.5));
        vec2 aspectUv = vec2((vUv.x - 0.5) * uScreenAspect + 0.5, vUv.y);
        
        vec2 rawGrid = aspectUv * uPatternScale;
        vec2 movement = uPatternDir * (progress * cycles);
        vec2 shiftedGrid = rawGrid - movement;
        
        float row = floor(shiftedGrid.y);
        float rowOffset = mod(abs(row), 2.0) * 0.5;
        
        vec2 gridUv = vec2(shiftedGrid.x - rowOffset, shiftedGrid.y);
        vec2 baseCell = floor(gridUv);
        vec2 cellCenter = baseCell + vec2(0.5);
        vec2 iconUv = (gridUv - cellCenter) * 2.2 + 0.5;

        if (iconUv.x > 0.0 && iconUv.x < 1.0 && iconUv.y > 0.0 && iconUv.y < 1.0) {
          vec4 iconTex = texture2D(uPatternTex, iconUv);
          col = mix(col, iconTex.rgb, iconTex.a * uPatternOpacity);
        }
      }

      gl_FragColor = vec4(col, 1.0);
    }
  `;

  function createShader(gl, type, source) {
    const shader = gl.createShader(type);
    gl.shaderSource(shader, source);
    gl.compileShader(shader);
    return shader;
  }

  const program = gl.createProgram();
  gl.attachShader(program, createShader(gl, gl.VERTEX_SHADER, vsSource));
  gl.attachShader(program, createShader(gl, gl.FRAGMENT_SHADER, fsSource));
  gl.linkProgram(program);
  gl.useProgram(program);

  const positionBuffer = gl.createBuffer();
  gl.bindBuffer(gl.ARRAY_BUFFER, positionBuffer);
  gl.bufferData(gl.ARRAY_BUFFER, new Float32Array([-1,-1, 1,-1, -1,1, -1,1, 1,-1, 1,1]), gl.STATIC_DRAW);

  const aPosition = gl.getAttribLocation(program, 'aPosition');
  gl.enableVertexAttribArray(aPosition);
  gl.vertexAttribPointer(aPosition, 2, gl.FLOAT, false, 0, 0);

  const uniforms = {};
  ['uResolution', 'uScreenAspect', 'uTime', 'uLoopDuration', 'uBgType', 'uBgColors', 'uBgPositions', 'uStopCount', 'uGradStart', 'uGradEnd', 'uBgAngle', 'uBgSpread', 'uEasingMode', 'uPatternOpacity', 'uPatternScale', 'uPatternSpeed', 'uPatternDir', 'uPatternTex', 'uHasPatternTex'].forEach(name => {
    uniforms[name] = gl.getUniformLocation(program, name);
  });

  const patternTexture = gl.createTexture();
  gl.activeTexture(gl.TEXTURE1);
  gl.bindTexture(gl.TEXTURE_2D, patternTexture);
  gl.pixelStorei(gl.UNPACK_PREMULTIPLY_ALPHA_WEBGL, false);
  gl.pixelStorei(gl.UNPACK_FLIP_Y_WEBGL, true);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_S, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_WRAP_T, gl.CLAMP_TO_EDGE);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MIN_FILTER, gl.LINEAR);
  gl.texParameteri(gl.TEXTURE_2D, gl.TEXTURE_MAG_FILTER, gl.LINEAR);
  gl.uniform1i(uniforms.uPatternTex, 1);

  let bgType = 1, selectedStopIndex = 0, gradStart = [0.15, 0.15], gradEnd = [0.85, 0.85], bgAngle = 45, bgSpread = 1.0, easingMode = 0, showHandles = true;
  let stopsState = [{ pos: 0.0, color: '#3b82f6' }, { pos: 1.0, color: '#1e3a8a' }];
  let isExporting = false, isPaused = false, animTime = 0, lastRealTime = performance.now(), loopDuration = 4.0, targetFps = 30;

  function getFormattedFilename(ext) {
    const rawInput = $('export-filename').value.trim() || 'BasicBG';
    const resString = `${canvas.width}x${canvas.height}`;
    return rawInput.includes('{resolution}') 
      ? `${rawInput.replace(/{resolution}/g, resString)}.${ext}`
      : `${rawInput}_${resString}.${ext}`;
  }

  function updateGradientRampUI() {
    stopsState.sort((a, b) => a.pos - b.pos);
    if (selectedStopIndex >= stopsState.length) selectedStopIndex = stopsState.length - 1;

    const track = $('gradient-ramp-track');
    track.style.background = `linear-gradient(90deg, ${stopsState.map(s => `${s.color} ${Math.round(s.pos * 100)}%`).join(', ')})`;
    track.querySelectorAll('.gradient-stop-handle').forEach(h => h.remove());

    stopsState.forEach((s, idx) => {
      const handle = document.createElement('div');
      handle.className = `gradient-stop-handle ${idx === selectedStopIndex ? 'active' : ''}`;
      handle.style.left = `${s.pos * 100}%`;
      handle.innerHTML = `<div class="stop-pin" style="background: ${s.color}"></div>`;

      handle.onpointerdown = (e) => {
        e.stopPropagation();
        selectedStopIndex = idx;
        updateGradientRampUI();

        const trackRect = track.getBoundingClientRect();
        function onPointerMove(me) {
          stopsState[selectedStopIndex].pos = parseFloat(Math.max(0, Math.min(1, (me.clientX - trackRect.left) / trackRect.width)).toFixed(3));
          updateGradientRampUI();
          sendGradientUniformsToWebGL();
        }
        function onPointerUp() {
          window.removeEventListener('pointermove', onPointerMove);
          window.removeEventListener('pointerup', onPointerUp);
        }
        window.addEventListener('pointermove', onPointerMove);
        window.addEventListener('pointerup', onPointerUp);
      };
      track.appendChild(handle);
    });

    $('stop-count-badge').innerText = `${stopsState.length} Stops`;
    $('btn-delete-stop').disabled = stopsState.length <= 2;
    syncActiveStopInputs();
  }

  function syncActiveStopInputs() {
    const activeStop = stopsState[selectedStopIndex];
    if (!activeStop) return;
    $('active-stop-color').value = activeStop.color;
    $('active-stop-hex').value = activeStop.color.toUpperCase();
    $('active-stop-pos').value = Math.round(activeStop.pos * 100);
    $('val-stop-pos').innerText = `${Math.round(activeStop.pos * 100)}%`;
  }

  function handleTrackClick(e) {
    const rect = $('gradient-ramp-track').getBoundingClientRect();
    const clickPos = Math.max(0, Math.min(1, (e.clientX - rect.left) / rect.width));
    if (stopsState.length >= 8) return alert("Maximum limit of 8 color stops reached.");

    stopsState.push({ pos: parseFloat(clickPos.toFixed(3)), color: interpolateGradientColorAt(clickPos) });
    stopsState.sort((a, b) => a.pos - b.pos);
    selectedStopIndex = stopsState.findIndex(s => Math.abs(s.pos - clickPos) < 0.001);
    updateGradientRampUI();
    sendGradientUniformsToWebGL();
  }

  function interpolateGradientColorAt(pos) {
    stopsState.sort((a, b) => a.pos - b.pos);
    if (!stopsState.length) return '#ffffff';
    if (pos <= stopsState[0].pos) return stopsState[0].color;
    if (pos >= stopsState[stopsState.length - 1].pos) return stopsState[stopsState.length - 1].color;

    for (let i = 0; i < stopsState.length - 1; i++) {
      if (pos >= stopsState[i].pos && pos <= stopsState[i+1].pos) {
        const c1 = parseHexColor(stopsState[i].color), c2 = parseHexColor(stopsState[i+1].color);
        const factor = (pos - stopsState[i].pos) / (stopsState[i+1].pos - stopsState[i].pos);
        const r = Math.round((c1[0] + (c2[0] - c1[0]) * factor) * 255);
        const g = Math.round((c1[1] + (c2[1] - c1[1]) * factor) * 255);
        const b = Math.round((c1[2] + (c2[2] - c1[2]) * factor) * 255);
        return `#${((1 << 24) + (r << 16) + (g << 8) + b).toString(16).slice(1)}`;
      }
    }
    return stopsState[0].color;
  }

  function addGradientStopAtCenter() {
    if (stopsState.length >= 8) return;
    stopsState.push({ pos: 0.5, color: interpolateGradientColorAt(0.5) });
    stopsState.sort((a, b) => a.pos - b.pos);
    selectedStopIndex = stopsState.findIndex(s => s.pos === 0.5);
    updateGradientRampUI();
    sendGradientUniformsToWebGL();
  }

  function deleteActiveStop() {
    if (stopsState.length <= 2) return;
    stopsState.splice(selectedStopIndex, 1);
    selectedStopIndex = Math.min(selectedStopIndex, stopsState.length - 1);
    updateGradientRampUI();
    sendGradientUniformsToWebGL();
  }

  function updateActiveStopColor(colorHex) { stopsState[selectedStopIndex].color = colorHex; updateGradientRampUI(); sendGradientUniformsToWebGL(); }
  function updateActiveStopHex(hex) { if (/^#[0-9A-F]{6}$/i.test(hex)) { stopsState[selectedStopIndex].color = hex; updateGradientRampUI(); sendGradientUniformsToWebGL(); } }
  function updateActiveStopPos(val) { stopsState[selectedStopIndex].pos = parseFloat((val / 100).toFixed(3)); updateGradientRampUI(); sendGradientUniformsToWebGL(); }
  function reverseGradientStops() { stopsState.forEach(s => s.pos = parseFloat((1.0 - s.pos).toFixed(3))); stopsState.sort((a, b) => a.pos - b.pos); updateGradientRampUI(); sendGradientUniformsToWebGL(); }
  function distributeGradientStops() {
    if (stopsState.length <= 1) return;
    stopsState.sort((a, b) => a.pos - b.pos).forEach((s, i) => s.pos = parseFloat((i / (stopsState.length - 1)).toFixed(3)));
    updateGradientRampUI();
    sendGradientUniformsToWebGL();
  }

  function parseHexColor(hex) {
    let c = hex.replace('#', '');
    if (c.length === 3) c = c.split('').map(x => x + x).join('');
    const num = parseInt(c, 16);
    return [((num >> 16) & 255) / 255, ((num >> 8) & 255) / 255, (num & 255) / 255];
  }

  function sendGradientUniformsToWebGL() {
    const flatColors = [], positions = [];
    stopsState.sort((a, b) => a.pos - b.pos);
    for (let i = 0; i < 8; i++) {
      if (i < stopsState.length) {
        const rgb = parseHexColor(stopsState[i].color);
        flatColors.push(...rgb);
        positions.push(stopsState[i].pos);
      } else { flatColors.push(0,0,0); positions.push(1.0); }
    }
    gl.uniform3fv(uniforms.uBgColors, new Float32Array(flatColors));
    gl.uniform1fv(uniforms.uBgPositions, new Float32Array(positions));
    gl.uniform1i(uniforms.uStopCount, stopsState.length);
    if (isPaused) renderFrame(animTime);
    renderOverlayHandles();
  }

  function renderOverlayHandles() {
    if (!showHandles || bgType === 0) { overlaySvg.style.display = 'none'; return; }
    overlaySvg.style.display = 'block';
    overlaySvg.innerHTML = '';

    const w = 512, h = 512;
    const p1 = [gradStart[0] * w, (1 - gradStart[1]) * h], p2 = [gradEnd[0] * w, (1 - gradEnd[1]) * h];
    overlaySvg.innerHTML = `<line class="handle-line" x1="${p1[0]}" y1="${p1[1]}" x2="${p2[0]}" y2="${p2[1]}" /><circle class="handle-circle" id="h-start" cx="${p1[0]}" cy="${p1[1]}" r="6" /><circle class="handle-circle" id="h-end" cx="${p2[0]}" cy="${p2[1]}" r="6" />`;
    attachHandleDrag('h-start', (x, y) => { gradStart = [Math.max(0, Math.min(1, x / w)), Math.max(0, Math.min(1, 1 - y / h))]; syncAngleFromPoints(); gl.uniform2f(uniforms.uGradStart, ...gradStart); sendGradientUniformsToWebGL(); });
    attachHandleDrag('h-end', (x, y) => { gradEnd = [Math.max(0, Math.min(1, x / w)), Math.max(0, Math.min(1, 1 - y / h))]; syncAngleFromPoints(); gl.uniform2f(uniforms.uGradEnd, ...gradEnd); sendGradientUniformsToWebGL(); });
  }

  function syncAngleFromPoints() {
    let angle = Math.atan2(gradEnd[1] - gradStart[1], gradEnd[0] - gradStart[0]) * (180 / Math.PI);
    bgAngle = Math.round(angle < 0 ? angle + 360 : angle);
    $('bg-angle').value = bgAngle; $('val-bg-angle').innerText = bgAngle + '°';
    gl.uniform1f(uniforms.uBgAngle, bgAngle);
  }

  function attachHandleDrag(id, onDrag) {
    const h = $(id);
    if (!h) return;
    h.onpointerdown = (e) => {
      e.preventDefault(); e.stopPropagation(); h.classList.add('active');
      const rect = overlaySvg.getBoundingClientRect();
      const move = me => onDrag(me.clientX - rect.left, me.clientY - rect.top);
      const up = () => { h.classList.remove('active'); window.removeEventListener('pointermove', move); window.removeEventListener('pointerup', up); };
      window.addEventListener('pointermove', move); window.addEventListener('pointerup', up);
    };
  }

  function toggleHandlesVisibility(visible) { showHandles = visible; renderOverlayHandles(); }

  function updateBgType(type) {
    const wrapper = $('gradient-controls-wrapper'), rowSolid = $('row-solid-color');
    if (type === 'solid') {
      bgType = 0; wrapper.style.display = 'none'; rowSolid.style.display = 'flex';
    } else {
      rowSolid.style.display = 'none'; wrapper.style.display = 'flex';
      bgType = 1;
    }
    gl.uniform1i(uniforms.uBgType, bgType);
    sendGradientUniformsToWebGL();
  }

  function updateSolidColor(hex) { $('val-solid-hex').innerText = hex.toUpperCase(); stopsState = [{ pos: 0, color: hex }]; sendGradientUniformsToWebGL(); }
  function updateBgAngle(deg) {
    bgAngle = parseFloat(deg); $('val-bg-angle').innerText = bgAngle + '°';
    gl.uniform1f(uniforms.uBgAngle, bgAngle);
    const rad = bgAngle * (Math.PI / 180);
    gradStart = [0.5 - Math.cos(rad) * 0.45, 0.5 - Math.sin(rad) * 0.45];
    gradEnd = [0.5 + Math.cos(rad) * 0.45, 0.5 + Math.sin(rad) * 0.45];
    gl.uniform2f(uniforms.uGradStart, ...gradStart); gl.uniform2f(uniforms.uGradEnd, ...gradEnd);
    sendGradientUniformsToWebGL();
  }

  function updateBgSpread(val) { bgSpread = parseFloat(val); $('val-bg-spread').innerText = bgSpread.toFixed(2); gl.uniform1f(uniforms.uBgSpread, bgSpread); sendGradientUniformsToWebGL(); }
  function updateGradientEasing(val) { easingMode = parseInt(val); gl.uniform1i(uniforms.uEasingMode, easingMode); sendGradientUniformsToWebGL(); }

  gl.uniform1i(uniforms.uBgType, 1);
  gl.uniform2f(uniforms.uGradStart, ...gradStart);
  gl.uniform2f(uniforms.uGradEnd, ...gradEnd);
  gl.uniform1f(uniforms.uBgAngle, 45.0);
  gl.uniform1f(uniforms.uBgSpread, 1.0);
  gl.uniform1i(uniforms.uEasingMode, 0);
  gl.uniform1f(uniforms.uPatternOpacity, 0.22);
  gl.uniform1f(uniforms.uPatternScale, 3.5);
  gl.uniform1f(uniforms.uPatternSpeed, 2.0);
  gl.uniform2f(uniforms.uPatternDir, 1.0, 0.0);
  gl.uniform1f(uniforms.uLoopDuration, loopDuration);
  gl.uniform1i(uniforms.uHasPatternTex, 0);

  updateGradientRampUI();
  sendGradientUniformsToWebGL();
  setResolution(512, 512);

  function setResolution(w, h) {
    canvas.width = w; canvas.height = h;
    gl.viewport(0, 0, w, h);
    gl.uniform2f(uniforms.uResolution, w, h);
    gl.uniform1f(uniforms.uScreenAspect, w / h);

    const c = $('pfp-container');
    c.style.width = w >= h ? '512px' : `${Math.round(512 * (w / h))}px`;
    c.style.height = w >= h ? `${Math.round(512 * (h / w))}px` : '512px';

    renderFrame(animTime);
    renderOverlayHandles();
  }

  function handleResolutionChange(val) {
    const ctrl = $('custom-res-controls');
    if (val === 'custom') ctrl.style.display = 'flex';
    else { ctrl.style.display = 'none'; setResolution(...val.split('x').map(Number)); }
  }

  function applyCustomResolution() {
    let w = parseInt($('custom-width').value) || 512, h = parseInt($('custom-height').value) || 512;
    setResolution(w % 2 === 0 ? w : w + 1, h % 2 === 0 ? h : h + 1);
  }

  function updateExportSettings() {
    loopDuration = parseFloat($('export-duration').value);
    targetFps = parseInt($('export-fps').value);
    gl.uniform1f(uniforms.uLoopDuration, loopDuration);
    animTime = animTime % loopDuration;
    if (isPaused) renderFrame(animTime);
  }

  function togglePause() {
    isPaused = !isPaused;
    $('pause-btn').innerText = isPaused ? "Resume Preview" : "Pause Preview";
    if (!isPaused) { lastRealTime = performance.now(); requestAnimationFrame(animLoop); }
  }

  function updatePatternDirection(dirKey) {
    const dirs = { 
      right: [1.0, 0.0], 
      'up-right': [1.0, 2.0], 
      up: [0.0, 2.0], 
      'up-left': [-1.0, 2.0], 
      left: [-1.0, 0.0], 
      'down-left': [-1.0, -2.0], 
      down: [0.0, -2.0], 
      'down-right': [1.0, -2.0] 
    };
    gl.uniform2f(uniforms.uPatternDir, ...(dirs[dirKey] || [1.0, 0.0]));
    if (isPaused) renderFrame(animTime);
  }

  function handlePatternUpload(e) {
    const file = e.target.files[0];
    if (!file) return;
    const reader = new FileReader();
    reader.onload = ev => {
      const img = new Image();
      img.onload = () => {
        gl.activeTexture(gl.TEXTURE1);
        gl.bindTexture(gl.TEXTURE_2D, patternTexture);
        gl.texImage2D(gl.TEXTURE_2D, 0, gl.RGBA, gl.RGBA, gl.UNSIGNED_BYTE, img);
        gl.uniform1i(uniforms.uHasPatternTex, 1);
        $('pattern-upload-label').innerText = `Pattern Loaded`;
        if (isPaused) renderFrame(animTime);
      };
      img.src = ev.target.result;
    };
    reader.readAsDataURL(file);
  }

  function updateUniform(name, val, displayId, suffix) {
    const num = name === 'uPatternSpeed' ? Math.round(parseFloat(val)) : parseFloat(val);
    $(displayId).innerText = (name === 'uPatternSpeed' ? num : num.toFixed(2)) + suffix;
    gl.uniform1f(uniforms[name], num);
    if (isPaused) renderFrame(animTime);
  }

  function renderFrame(t) { gl.uniform1f(uniforms.uTime, t); gl.drawArrays(gl.TRIANGLES, 0, 6); }

  function animLoop() {
    if (isExporting || isPaused) return;
    const now = performance.now();
    animTime = (animTime + (now - lastRealTime) / 1000) % loopDuration;
    lastRealTime = now;
    renderFrame(animTime);
    requestAnimationFrame(animLoop);
  }
  requestAnimationFrame(animLoop);

  async function triggerExport() {
    const fmt = $('export-format').value;
    fmt === 'zip' ? await exportPNGSequence() : await exportInBrowserVideo(fmt);
  }

  async function exportInBrowserVideo(format) {
    if (typeof VideoEncoder === "undefined") return alert("WebCodecs API is not supported in this browser.");
    isExporting = true; toggleUIControls(false);

    const totalFrames = Math.round(loopDuration * targetFps);
    const w = canvas.width % 2 === 0 ? canvas.width : canvas.width + 1;
    const h = canvas.height % 2 === 0 ? canvas.height : canvas.height + 1;
    let muxer, videoEncoder;

    try {
      if (format === 'mp4') {
        muxer = new Mp4Muxer.Muxer({ target: new Mp4Muxer.ArrayBufferTarget(), video: { codec: 'avc', width: w, height: h }, fastStart: 'in-memory' });
        videoEncoder = new VideoEncoder({ 
          output: (c, m) => muxer.addVideoChunk(c, m), 
          error: e => { console.error("Encoder error:", e); alert("Export failed: " + e.message); cleanupExportUI(); } 
        });
        videoEncoder.configure({ codec: 'avc1.640033', width: w, height: h, bitrate: 8_000_000, framerate: targetFps });
      } else {
        muxer = new WebmMuxer.Muxer({ target: new WebmMuxer.ArrayBufferTarget(), video: { codec: 'V_VP9', width: w, height: h, frameRate: targetFps } });
        videoEncoder = new VideoEncoder({ 
          output: (c, m) => muxer.addVideoChunk(c, m), 
          error: e => { console.error("Encoder error:", e); alert("Export failed: " + e.message); cleanupExportUI(); } 
        });
        videoEncoder.configure({ codec: 'vp09.00.10.08', width: w, height: h, bitrate: 8_000_000, framerate: targetFps });
      }

      for (let i = 0; i < totalFrames; i++) {
        if (!isExporting) break;

        while (videoEncoder.encodeQueueSize > 4) {
          await new Promise(r => setTimeout(r, 5));
        }

        updateProgressBar(Math.round(((i + 1) / totalFrames) * 100), `Rendering frame ${i + 1} of ${totalFrames}...`);
        renderFrame((i / totalFrames) * loopDuration);
        gl.flush();

        const timestampUs = Math.round((i / targetFps) * 1_000_000);
        const nextTimestampUs = Math.round(((i + 1) / targetFps) * 1_000_000);
        const frameDurationUs = nextTimestampUs - timestampUs;

        const videoFrame = new VideoFrame(canvas, { timestamp: timestampUs, duration: frameDurationUs });
        videoEncoder.encode(videoFrame, { keyFrame: i % (targetFps * 2) === 0 });
        videoFrame.close();

        await new Promise(r => requestAnimationFrame(r));
      }

      if (isExporting) {
        updateProgressBar(100, "Finalizing file compilation...");
        await videoEncoder.flush();
        muxer.finalize();
        downloadBlob(new Blob([muxer.target.buffer], { type: format === 'mp4' ? 'video/mp4' : 'video/webm' }), getFormattedFilename(format));
      }
    } catch (err) {
      console.error("Export process failed:", err);
      alert("Export error occurred: " + err.message);
    } finally {
      cleanupExportUI();
    }
  }

  async function exportPNGSequence() {
    isExporting = true; toggleUIControls(false);
    const totalFrames = Math.round(loopDuration * targetFps);
    const zip = new JSZip(), folder = zip.folder("bg_frames");

    try {
      for (let i = 0; i < totalFrames; i++) {
        if (!isExporting) break;
        updateProgressBar(Math.round(((i + 1) / totalFrames) * 100), `Rendering PNG Frame ${i + 1} of ${totalFrames}...`);
        renderFrame((i / totalFrames) * loopDuration);
        folder.file(`bg_frame_${String(i + 1).padStart(4, '0')}.png`, canvas.toDataURL('image/png').replace(/^data:image\/png;base64,/, ""), { base64: true });
        if (i % 5 === 0) await new Promise(r => setTimeout(r, 0));
      }

      if (isExporting) {
        updateProgressBar(100, "Compressing frames to ZIP archive...");
        downloadBlob(await zip.generateAsync({ type: "blob" }), getFormattedFilename('zip'));
      }
    } catch (err) {
      console.error("ZIP Export failed:", err);
      alert("PNG sequence export failed: " + err.message);
    } finally {
      cleanupExportUI();
    }
  }

  function updateProgressBar(percent, text) {
    $('progress-container').style.display = 'flex';
    $('progress-fill').style.width = `${percent}%`;
    $('progress-text').innerText = text;
  }

  function toggleUIControls(enabled) { $('export-btn').disabled = !enabled; $('pause-btn').disabled = !enabled; }
  function cleanupExportUI() {
    $('progress-container').style.display = 'none';
    toggleUIControls(true); isExporting = false;
    renderFrame(animTime);
    if (!isPaused) { lastRealTime = performance.now(); requestAnimationFrame(animLoop); }
  }

  function downloadBlob(blob, filename) {
    const link = document.createElement('a');
    link.download = filename;
    link.href = URL.createObjectURL(blob);
    link.click();
    URL.revokeObjectURL(link.href);
  }
</script>
</body>
</html>
