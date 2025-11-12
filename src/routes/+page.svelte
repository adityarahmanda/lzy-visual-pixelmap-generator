<script lang="ts">
import { onMount } from "svelte";

let canvasRef: HTMLCanvasElement;
let ctx: CanvasRenderingContext2D | null = null;

// ---- CONFIG ----
let width:number;
let height:number;
let gridPerfectSquare:boolean;
let rows:number;
let showLabel:boolean;
let labelText:string;
let labelLeftColor:string;
let labelRightColor:string;
let labelSize:number;
let showBackground:boolean;
let showLogo:boolean;
let logoSize:number;
let gridColor:string;
let gridThickness:number;
resetAll();

function resetAll() {
  width = 1920;
  height = 1080;
  gridPerfectSquare = false;
  rows = 12;
  showLabel = true;
  labelText = "A1";
  labelLeftColor = "#ffffff";
  labelRightColor = "#000000";
  labelSize = 1.0;
  showBackground = true;
  showLogo = true;
  logoSize = 1.0;
  gridColor = "#ffffff";
  gridThickness = 1.5;
  resizeAndDraw();
}

function drawCanvas() {
  if (!canvasRef || !ctx) return;
  const { width, height } = ctx.canvas;
  ctx.clearRect(0, 0, width, height);

  if (showBackground) {
    const bg = new Image();
    bg.src = "./background.png";
    bg.onload = () => {
      if (!ctx) return;

      // cover (envelope) the canvas, maintaining aspect ratio
      const canvasRatio = width / height;
      const imgRatio = bg.naturalWidth / bg.naturalHeight;

      let drawWidth, drawHeight, offsetX, offsetY;

      if (imgRatio > canvasRatio) {
        // image wider → crop sides
        drawHeight = height;
        drawWidth = height * imgRatio;
        offsetX = (width - drawWidth) / 2;
        offsetY = 0;
      } else {
        // image taller → crop top/bottom
        drawWidth = width;
        drawHeight = width / imgRatio;
        offsetX = 0;
        offsetY = (height - drawHeight) / 2;
      }

      ctx.drawImage(bg, offsetX, offsetY, drawWidth, drawHeight);
      drawGridAndLogo();
    };
  } else {
    drawGridAndLogo();
  }
}

function drawGridAndLogo() {
  if (!canvasRef || !ctx) return;
  const { width, height } = ctx.canvas;

  // prepare grid
  const cellHeight = height / rows;
  const cols = Math.floor(width / cellHeight);
  const cellWidth = gridPerfectSquare ? cellHeight : width / cols;
  const gridWidth = cols * cellWidth;
  const gridHeight = rows * cellHeight;

  const parent = canvasRef.parentElement as HTMLDivElement | null;
  let parentWidth = width;
  let parentHeight = height;
  if (parent) {
    const canvasRect = canvasRef.getBoundingClientRect();
    const scaleX = width / canvasRect.width;
    const scaleY = height / canvasRect.height;

    const parentRect = parent.getBoundingClientRect();
    parentWidth = parentRect.width * scaleX;
    parentHeight = parentRect.height * scaleY;
  }

  const startX = (width - gridWidth) / 2 + (parentWidth - width) / 2;
  const startY = (height - gridHeight) / 2 + (parentHeight - height) / 2;

  ctx.save();
  ctx.strokeStyle = gridColor;
  ctx.lineWidth = gridThickness;

  // draw horizontal lines
  for (let r = 0; r <= rows; r++) {
    const y = startY + r * cellHeight;
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(width, y);
    ctx.stroke();
  }

  // draw vertical lines
  for (let c = 0; c <= cols; c++) {
    const x = startX + c * cellWidth;
    ctx.beginPath();
    ctx.moveTo(x, 0);
    ctx.lineTo(x, gridHeight);
    ctx.stroke();
  }

  // label numbers
  const fontSize = cellHeight * 0.4;
  ctx.font = `${fontSize}px Sora`;
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";
  ctx.fillStyle = gridColor;
  const metrics = ctx.measureText("0");
  const yOffset = metrics.actualBoundingBoxDescent / 2;
  const topRowY = startY + cellHeight / 2 + yOffset;
  const bottomRowY = startY + (rows - 1) * cellHeight + cellHeight / 2 + yOffset;
  const centerCol = Math.floor(cols / 2);

  for (let c = 0; c < cols; c++) {
    const x = startX + c * cellWidth + cellWidth / 2;
    const distFromCenter = Math.abs(c - centerCol);

    let label: number;
    if (cols % 2 === 0) {
      const leftCenter = cols / 2 - 1;
      const rightCenter = cols / 2;
      const mid = (leftCenter + rightCenter) / 2;
      label = Math.max(1, Math.round(Math.abs(c - mid)));
    } else {
      // Odd → use actual center column
      label = Math.abs(distFromCenter);
    }

    ctx.fillText(label.toString(), x, topRowY);
    ctx.fillText(label.toString(), x, bottomRowY);
  }

  if (showLabel) {
    const gridSlice = 10;
    const gridSliceSize = gridWidth / gridSlice;
    const labelY = height / 2;
    const fontSize = height * (1 / 3) * labelSize;
    ctx.font = `bold ${fontSize}px Sora`;
    ctx.fillStyle = labelLeftColor;
    const metrics = ctx.measureText("0");
    const yOffset = metrics.actualBoundingBoxDescent / 2;
    ctx.fillText(labelText, startX + 1.5 * gridSliceSize, labelY + yOffset);
    ctx.fillStyle = labelRightColor;
    ctx.fillText(labelText, startX + (gridSlice - 1.5) * gridSliceSize, labelY + yOffset);
  }

  // ---- DRAW LOGO ----
  if (showLogo) {
    const img = new Image();
    img.src = "./logo.png";
    img.onload = () => {
      if (!ctx) return;
      const centerX = startX + gridWidth / 2;
      const centerY = startY + gridHeight / 2;

      const maxLogoHeight = (rows - 6) * cellHeight; // leave 2-cell margin top & bottom
      const aspect = img.naturalWidth / img.naturalHeight;
      const imgHeight = Math.min(maxLogoHeight, gridHeight - 4 * cellHeight) * logoSize;
      const imgWidth = imgHeight * aspect;

      ctx.drawImage(
        img,
        centerX - imgWidth / 2,
        centerY - imgHeight / 2,
        imgWidth,
        imgHeight
      );
    };
  }

  ctx.restore();
}

function resizeAndDraw() {
  if (!canvasRef || !ctx) return;
  canvasRef.width = width;
  canvasRef.height = height;
  canvasRef.style.width = `${width}px`;
  canvasRef.style.height = `${height}px`;
  drawCanvas();
}

function saveCanvas() {
  if (!canvasRef) return;
  const link = document.createElement("a");
  link.download = "test-card.png";
  link.href = canvasRef.toDataURL("image/png");
  link.click();
}

onMount(() => {
  ctx = canvasRef.getContext("2d");
  resizeAndDraw();
});

// reactive redraw whenever configs change
$: {
  gridPerfectSquare;
  width;
  height;
  rows;
  showLabel;
  labelText;
  labelLeftColor;
  labelRightColor;
  labelSize;
  showBackground;
  showLogo;
  logoSize;
  gridColor;
  gridThickness;
  resizeAndDraw();
}
</script>

<svelte:head>
	<title>LZY Visual Pixelmap Generator</title>
</svelte:head>

<section>
   <div class="pixelmap-app">
      <header class="app-title-header">
         <h1>LZY Visual Pixelmap Generator</h1>
      </header>
      <div class="canvas-preview-container">
        <canvas
          bind:this={canvasRef}
          style="width: {width}px; height: {height}px;"
          width={width}
          height={height} 
        ></canvas>
      </div>
      <div class="controls-main-container">
         <div class="controls-grid">
            <div class="control-column">
               <div class="control-item">
                  <label>Image Output Size (1x):</label>
                  <div class="control-item-row">
                    <input id="widthInput" inputmode="numeric" aria-label="Image Width" type="text" bind:value={width}><span>X</span>
                    <input id="heightInput" inputmode="numeric" aria-label="Image Height" type="text" bind:value={height}></div>
               </div>
               <div class="control-item">
                <label for="gridRowsSlider">Grid Rows: ({rows})</label>
                <input id="gridRowsSlider" min="4" max="48" type="range" bind:value={rows}>
              </div>
               <div class="control-item">
                <label for="gridThicknessSlider">Grid Thickness [px @1x]: ({gridThickness})</label>
                <input id="gridThicknessSlider" min="0" max="5" step="0.1" type="range" bind:value={gridThickness}>
              </div>
               <div class="control-item">
                <label class="toggle-switch-label"><span>Grid Perfect Square:</span>
                  <label class="switch"><input id="stretchCell" type="checkbox" bind:checked={gridPerfectSquare}>
                    <span class="slider-toggle round"></span>
                  </label>
                </label>
              </div>
               <div class="control-item">
                <div style="display: flex; align-items: center; justify-content: space-between;">
                  <label for="gridColorPicker" style="margin-right: 10px;">Grid Color:</label>
                  <input id="gridColorPicker" type="color" bind:value={gridColor}>
                </div>
              </div>
            </div>
            <div class="control-column">
              <div class="control-item">
                <label class="toggle-switch-label">
                  <span>Background:</span>
                  <label class="switch">
                    <input id="backgroundToggle" type="checkbox" bind:checked={showBackground}>
                    <span class="slider-toggle round"></span>
                  </label>
                </label>
              </div>
              <div class="control-item">
                <label class="toggle-switch-label">
                  <span>Logo:</span>
                  <label class="switch">
                    <input id="logoToggle" type="checkbox" bind:checked={showLogo}>
                    <span class="slider-toggle round"></span>
                  </label>
                </label>
              </div>
              <div class="control-item">
                <label for="logoSizeSlider">Logo Size: ({logoSize})</label>
                <input id="logoSizeSlider" min="0.1" max="3" step="0.1" type="range" bind:value={logoSize}>
              </div>
              <div class="control-item">
                <label class="toggle-switch-label"><span>Label:</span>
                  <label class="switch"><input id="labelToggle" type="checkbox" bind:checked={showLabel}>
                    <span class="slider-toggle round"></span>
                  </label>
                </label>
              </div>
              <div class="control-item">
                <label for="labelText">Label Text:</label>
                <input id="labelText" type="text" bind:value={labelText}>
              </div>
              <div class="control-item">
                <label for="logoSizeSlider">Label Size: ({labelSize})</label>
                <input id="logoSizeSlider" min="0.1" max="3" step="0.1" type="range" bind:value={labelSize}>
              </div>
              <div class="control-item">
                <div style="display: flex; align-items: center; justify-content: space-between;">
                  <label for="labelLeftColorPicker" style="margin-right: 10px;">Label Left Color:</label>
                  <input id="labelLeftColorPicker" type="color" bind:value={labelLeftColor}>
                </div>
              </div>
              <div class="control-item">
                <div style="display: flex; align-items: center; justify-content: space-between;">
                  <label for="labelRightColorPicker" style="margin-right: 10px;">Label Right Color:</label>
                  <input id="labelRightColorPicker" type="color" bind:value={labelRightColor}>
                </div>
              </div>
            </div>
         </div>
         <div class="action-buttons-container">
          <button class="action-button reset-button" on:click={resetAll} style="margin-right: 10px;">Reset All</button>
          <button class="action-button" on:click={saveCanvas}>Save Image ({width}x{height})</button></div>
      </div>
   </div>
</section>
