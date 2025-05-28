<template>
  <div class="app">
    <div class="form-container">
      <input v-model="text" placeholder="Enter rank text..." class="input dark-input" />

      <canvas ref="canvas" class="preview-canvas"></canvas>
      <img :src="imageSrc" alt="Preview" class="preview-img"
        :style="{ backgroundColor: bgColor, borderColor: borderColor }" />
      
      <div class="color-pickers">
        <label>
          Background:
          <input type="color" v-model="bgColor" class="color-picker" />
        </label>
        <label> 
          Background HEX:
          <input 
            type="text" 
            v-model="bgColorHex" 
            @input="handleBgColorHexInput" 
            class="input dark-input" 
            style="width: 120px; margin-left: 0.5rem; font-size: 1rem; padding: 0.25rem;" 
            placeholder="#RRGGBB"
          />
        </label>
        <label>
          Border:
          <input type="color" v-model="borderColor" class="color-picker" />
          <input type="checkbox" v-model="showBorder" style="margin-left: 0.5rem;" /> Show 
        </label>
        <label>
          Shadow:
          <input type="color" v-model="shadowColor" class="color-picker" />
          <input type="checkbox" v-model="showShadow" style="margin-left: 0.5rem;" /> Show
        </label>
        <label>
          Presets:
          <select v-model="selectedPreset" @change="applyPreset" class="dark-select" size="10">
            <option disabled value="">Select preset</option>
            <option v-for="(_, name) in presets" :key="name" :value="name">{{ name }}</option>
          </select>
        </label>
      </div>
      <button @click="downloadImage" class="dark-button">Download Image</button> 
    </div>
  </div>
</template>

<script setup>
import { ref, watch, onMounted, nextTick } from 'vue';

// --- Reactive State ---
const text = ref('Rank+*-Test'); 
const canvas = ref(null);
const imageSrc = ref('');
const bgColor = ref('#282828');
const borderColor = ref('#a0a0a0'); 
const shadowColor = ref('#505050');
const selectedPreset = ref('');
const showBorder = ref(true); 
const showShadow = ref(true); 
const fontLoaded = ref(false);

const bgColorHex = ref(bgColor.value);

// --- Constants and Configuration ---
const tileSize = 8;
const padding = 1; 
const spacing = 1;

const fontImage = new Image();
fontImage.crossOrigin = "Anonymous"; 
let charWidths = {}; 

const saturation = 0.08; 
const presets = {
  Classic: { bg: '#282828', border: '#a0a0a0' },
  Emerald: { bg: '#003e2f', border: '#00ffba' },
  Gold: { bg: '#3b2c00', border: '#ffcc00' },
  Nether: { bg: '#2b0f0f', border: '#ff3b3b' },
  Ice: { bg: '#0f2b3b', border: '#3bafff' },
  Diamond: { bg: '#0f3b3b', border: '#3bffff' },
  Ruby: { bg: '#3b0f0f', border: '#ff3b6b' },
  Amethyst: { bg: '#2b0f3b', border: '#a03bff' },
  Obsidian: { bg: '#0f0f2b', border: '#3b3bff' },
  Sandstone: { bg: '#3b2b0f', border: '#ffcc66' },
  Lapis: { bg: '#0f0f3b', border: '#3b6bff' },
  Ender: { bg: '#1a0f2b', border: '#7f3bff' },
  Prismarine: { bg: '#0f3b2b', border: '#3bffcc' },
  Copper: { bg: '#3b1f0f', border: '#ff9966' },
  Glowstone: { bg: '#3b3b0f', border: '#ffff66' },
  Crimson: { bg: '#3b0f1f', border: '#ff6699' },
  Warped: { bg: '#0f3b1f', border: '#66ff99' }
};

for (const name in presets) {
  const preset = presets[name];
  preset.shadow = adjustHSL(preset.bg, saturation); 
}

function adjustHSL(colorHex, lightnessAdjustment) {
  const r = parseInt(colorHex.slice(1, 3), 16) / 255;
  const g = parseInt(colorHex.slice(3, 5), 16) / 255;
  const b = parseInt(colorHex.slice(5, 7), 16) / 255;

  const max = Math.max(r, g, b);
  const min = Math.min(r, g, b);
  const l = (max + min) / 2;

  const s = max === min ? 0 : l < 0.5 ? (max - min) / (max + min) : (max - min) / (2 - max - min);

  const h = (() => {
    if (max === min) return 0;
    if (max === r) return ((g - b) / (max - min) + (g < b ? 6 : 0)) / 6;
    if (max === g) return ((b - r) / (max - min) + 2) / 6;
    return ((r - g) / (max - min) + 4) / 6;
  })();

  const adjustedL = Math.min(Math.max(l + lightnessAdjustment, 0), 1);
  const q = adjustedL < 0.5 ? adjustedL * (1 + s) : adjustedL + s - adjustedL * s;
  const p = 2 * adjustedL - q;

  const toRGB = t => {
    if (t < 0) t += 1; if (t > 1) t -= 1;
    if (t < 1 / 6) return p + (q - p) * 6 * t;
    if (t < 1 / 2) return q;
    if (t < 2 / 3) return p + (q - p) * (2 / 3 - t) * 6;
    return p;
  };

  const newR = Math.round(toRGB(h + 1 / 3) * 255);
  const newG = Math.round(toRGB(h) * 255);
  const newB = Math.round(toRGB(h - 1 / 3) * 255);

  return `#${newR.toString(16).padStart(2, '0')}${newG.toString(16).padStart(2, '0')}${newB.toString(16).padStart(2, '0')}`;
}


const applyPreset = () => {
  const preset = presets[selectedPreset.value];
  if (preset) {
    bgColor.value = preset.bg;
    borderColor.value = preset.border;
    shadowColor.value = preset.shadow;
  }
};

watch(bgColor, (newColor) => {
  if (newColor !== bgColorHex.value) {
    bgColorHex.value = newColor;
  }
});

function handleBgColorHexInput() {
  let newHex = bgColorHex.value.trim();
  if (!newHex.startsWith('#')) {
    newHex = '#' + newHex;
  }
  if (/^#([0-9A-Fa-f]{3}){1,2}$/.test(newHex)) {
    if (bgColor.value !== newHex) {
      bgColor.value = newHex; 
    }
  } else {
    console.warn("Invalid hex input for background:", bgColorHex.value);
  }
}

const charToIndex = char => {
  const code = char.charCodeAt(0);
  return code >= 32 && code <= 127 ? code - 32 : 0; 
};

const analyzeCharWidths = () => {
  console.log("Analyzing char widths (original logic)...");
  if (!fontImage.complete || fontImage.naturalWidth === 0) {
    console.error("Font image not loaded or zero size for analyzeCharWidths.");
    for (let i = 0; i < 96; i++) { charWidths[i] = { width: tileSize, offsetX: 0, isPlaceholder: false }; }
    return;
  }
  const offCanvas = document.createElement('canvas');
  offCanvas.width = 128; 
  offCanvas.height = 128; 
  const ctx = offCanvas.getContext('2d');
  if (!ctx) { console.error("Failed to get 2D context for char width analysis."); return; }
  ctx.drawImage(fontImage, 0, 0); 

  const symbolPlaceholderIndices = [10, 11, 13]; 

  for (let i = 0; i < 96; i++) { 
    const sx = (i % 16) * tileSize, sy = Math.floor(i / 16) * tileSize;
    let minX = tileSize, maxX = 0, hasPixel = false;
    
    if (sx + tileSize > offCanvas.width || sy + tileSize > offCanvas.height) {
        console.warn(`Char index ${i} tile (sx=${sx}, sy=${sy}) is out of 128x128 offscreen canvas bounds.`);
        hasPixel = false; 
    } else {
        try {
            const imageData = ctx.getImageData(sx, sy, tileSize, tileSize);
            const data = imageData.data;
            for (let y = 0; y < tileSize; y++) {
                for (let x = 0; x < tileSize; x++) {
                    if (data[(y * tileSize + x) * 4 + 3] > 0) { 
                        if (x < minX) minX = x; if (x > maxX) maxX = x; hasPixel = true;
                    }
                }
            }
        } catch (e) {
            console.error(`Error getting ImageData for char index ${i} at sx=${sx}, sy=${sy}:`, e);
            hasPixel = false; 
        }
    }

    if (!hasPixel) {
        if (i === 0) { 
            charWidths[i] = { width: Math.floor(tileSize / 2), offsetX: 0, isPlaceholder: false };
        } else if (symbolPlaceholderIndices.includes(i)) { 
            console.warn(`Original logic: Symbol index ${i} (*, +, -) is blank. Rendering as white placeholder.`);
            charWidths[i] = { 
                width: Math.floor(tileSize * 0.7), 
                offsetX: Math.floor(tileSize * 0.15), 
                isPlaceholder: true 
            };
        } else { 
            charWidths[i] = { width: 0, offsetX: 0, isPlaceholder: false }; 
        }
    } else { 
        charWidths[i] = { width: maxX - minX + 1, offsetX: minX, isPlaceholder: false };
    }
  }
  console.log("Char widths analyzed (original logic):", charWidths);
};

const draw = () => {
  if (!canvas.value || !fontLoaded.value) {
    console.warn("Draw called but canvas or font not ready.");
    return;
  }
  console.log("Drawing image for text (original logic):", text.value);
  const ctx = canvas.value.getContext('2d');
  if (!ctx) { console.error("Failed to get 2D context."); return; }
  ctx.imageSmoothingEnabled = false;

  const chars = text.value.toLowerCase().split(''); 
  let currentTotalWidth = padding + 1; 

  for (const char of chars) {
    const charIdx = charToIndex(char);
    const charInfo = charWidths[charIdx] || { width: tileSize, offsetX: 0, isPlaceholder: false };
    currentTotalWidth += charInfo.width + spacing;
  }
  
  const finalWidth = currentTotalWidth - spacing + (padding +1) ; 
  const canvasHeight = tileSize + 1; 

  canvas.value.width = finalWidth; 
  canvas.value.height = canvasHeight; 

  ctx.fillStyle = bgColor.value; 
  ctx.fillRect(0, 0, finalWidth, canvasHeight);

  if (showBorder.value) { 
    ctx.strokeStyle = borderColor.value; 
    ctx.lineWidth = 1; 
    ctx.strokeRect(0, 0, finalWidth, canvasHeight); 
  }

  let cursorX = padding + 1; 

  for (const char of chars) {
    const i = charToIndex(char);
    const charInfo = charWidths[i] || { width: tileSize, offsetX: 0, isPlaceholder: false }; 
    const drawY = 0; 

    if (charInfo.isPlaceholder) { 
        if (charInfo.width > 0) { 
            if (showShadow.value) {
                ctx.fillStyle = shadowColor.value;
                ctx.fillRect(cursorX + 1, drawY + 1, charInfo.width, tileSize); 
            }
            ctx.fillStyle = '#FFFFFF'; 
            ctx.fillRect(cursorX, drawY, charInfo.width, tileSize);
        }
    } else {
        if (charInfo.width > 0) { 
            const tileSpriteX = (i % 16) * tileSize + charInfo.offsetX;
            const tileSpriteY = Math.floor(i / 16) * tileSize;

            if (showShadow.value) {
                const shadowCanvas = colorize(fontImage, shadowColor.value);
                if (shadowCanvas) {
                    ctx.drawImage(shadowCanvas, tileSpriteX, tileSpriteY, charInfo.width, tileSize,
                                  cursorX + 1, drawY, charInfo.width, tileSize); 
                }
            }
            ctx.drawImage(fontImage, tileSpriteX, tileSpriteY, charInfo.width, tileSize,
                          cursorX, drawY, charInfo.width, tileSize);
        }
    }
    cursorX += charInfo.width + spacing;
  }
  try {
    imageSrc.value = canvas.value.toDataURL('image/png');
    console.log("Image drawn. Data URL length:", imageSrc.value.length);
  } catch (e) { console.error("Error converting canvas to data URL:", e); }
};

function colorize(image, colorHex) {
  const r = parseInt(colorHex.slice(1, 3), 16) / 255;
  const g = parseInt(colorHex.slice(3, 5), 16) / 255;
  const b = parseInt(colorHex.slice(5, 7), 16) / 255;
  const imageSize = image.width; 
  const offscreen = new OffscreenCanvas(imageSize, imageSize); 
  const ctx = offscreen.getContext("2d");
  ctx.drawImage(image, 0, 0);
  const imageData = ctx.getImageData(0, 0, imageSize, imageSize);
  const data = imageData.data;
  for (let i = 0; i < data.length; i += 4) {
    data[i + 0] *= r; data[i + 1] *= g; data[i + 2] *= b;
  }
  ctx.putImageData(imageData, 0, 0);
  return offscreen;
}

function downloadImage() {
  if (!imageSrc.value) { alert("Please generate an image first."); return; }
  const link = document.createElement('a');
  link.href = imageSrc.value;
  link.download = `${text.value.replace(/[^a-z0-9_+\-*]/gi, '_') || 'rank'}.png`;
  document.body.appendChild(link); link.click(); document.body.removeChild(link);
  console.log("Download initiated for:", link.download);
}

watch([text, bgColor, borderColor, shadowColor, showBorder, showShadow], () => {
  if (fontLoaded.value && canvas.value) { draw(); }
}, { deep: true });

onMounted(() => {
  console.log("Component mounted (original logic for font loading).");
  nextTick(() => { 
    if (!canvas.value) {
      console.error("Canvas ref NULL after nextTick.");
      alert("Error: Canvas element not found."); return;
    }
    fontImage.onload = () => {
      console.log("Font image loaded (original onMounted):", fontImage.src);
      fontLoaded.value = true; 
      analyzeCharWidths(); 
      draw();              
    };
    fontImage.onerror = () => {
        console.error("CRITICAL: Error loading font image (original onMounted):", fontImage.src);
        alert(`Error: Could not load font image from ${fontImage.src}. Placeholders might be used.`);
        fontLoaded.value = true; 
        analyzeCharWidths(); 
        draw();
    };
    const pathsToTry = [
      '/minecraft-rank-generator/ascii.png', 
      '/ascii.png',
      'ascii.png' 
    ];
    let currentPathIndex = 0;
    function loadImageAttempt() {
      if (currentPathIndex >= pathsToTry.length) {
        console.error("CRITICAL: Failed to load font image from all attempted paths (original onMounted style):", pathsToTry);
        fontImage.onerror(); 
        return;
      }
      const currentPath = pathsToTry[currentPathIndex];
      console.log("Attempting to load font image from (original onMounted style):", currentPath);
      fontImage.src = currentPath;
    }
    loadImageAttempt();
  });
});

</script>

<style>
/* Component-specific styles for App.vue */
/* These styles will work in conjunction with the global style.css */

.app {
  width: 100%; 
  padding: 0; 
  box-sizing: border-box;
}

.form-container {
  display: flex;
  flex-direction: column;
  gap: 1.5rem;
  align-items: center; 
  background: #2a2a2a; 
  padding: 2rem;
  border-radius: 8px;
  box-shadow: 0 4px 10px rgba(0, 0, 0, 0.5);
  border: 1px solid #4a4a4a;
  width: 100%; 
  max-width: 600px; 
  color: white; 
  text-align: left; 
}

.input.dark-input { 
  padding: 0.5rem;
  font-size: 1.2rem;
  width: 100%; 
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  background: #1e1e2e; 
  color: #e0e0ff; 
  box-sizing: border-box; 
  font-family: inherit; 
}

.input.dark-input::placeholder {
  color: #a0a0c0;
}

.form-container .color-pickers input[type="text"].dark-input {
    width: 120px !important; 
    margin-left: 0.5rem !important;
    font-size: 1rem !important;
    padding: 0.25rem !important;
}

.color-pickers { 
  display: flex;
  flex-direction: column; 
  gap: 1rem;
  align-items: stretch; 
  width: 100%; 
}

.color-pickers label { 
    display: flex;
    align-items: center; 
    justify-content: space-between; 
    font-size: 0.9rem; 
    width: 100%;
}

.color-pickers label > span:first-child { 
    margin-right: 0.5rem; 
}

.color-picker { 
  margin-left: 0.5rem; 
  border: 1px solid #4a4a4a; 
  background-color: #1e1e2e; 
  border-radius: 4px;
  padding: 2px; 
  height: 30px; 
  width: 50px; 
  cursor: pointer;
  -webkit-appearance: none; 
  -moz-appearance: none;
  appearance: none;
}
.color-picker::-webkit-color-swatch-wrapper { padding: 0; }
.color-picker::-webkit-color-swatch { border: none; border-radius: 3px;}
.color-picker::-moz-color-swatch { border: none; border-radius: 3px;}

.dark-select { 
  margin-left: 0.5rem; 
  padding: 0.4rem;
  border: 1px solid #4a4a4a;
  border-radius: 4px;
  background: #1e1e2e;
  color: #e0e0ff;
  font-family: inherit; 
}

.preview-canvas {
  display: none;
}

.preview-img {
  image-rendering: pixelated;
  width: auto;
  height: 128px; 
  margin-bottom: 1rem; 
}

.dark-button { 
  padding: 0.6rem 1.2rem; 
  background: #3a3a5a;   
  border: 1px solid #5a5a8a; 
  color: #e0e0ff;        
  border-radius: 6px;     
  transition: background-color 0.3s, border-color 0.25s; 
  width: 100%; 
  box-sizing: border-box;
  /* font-family, font-size, font-weight, cursor are inherited from global button style */
}

.dark-button:hover {
  background: #4a4a7a; 
  border-color: #646cff; 
}

.color-pickers input[type="checkbox"] {
  margin-left: 0.5rem; 
  margin-right: 0.25rem; 
  vertical-align: middle; 
}
</style>

