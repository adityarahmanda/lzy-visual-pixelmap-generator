<script lang="ts">
import { onMount } from "svelte";

interface Grid {
  gridWidth: number;
  gridHeight: number;
  cellWidth: number;
  cellHeight: number;
  cols: number;
  rows: number;
  startX: number;
  startY: number;
}

let canvasRef: HTMLCanvasElement;
let backgroundImage: HTMLImageElement;
let logoImage: HTMLImageElement;
let ctx: CanvasRenderingContext2D | null = null;

// ---- CONFIG ----
let width:number;
let height:number;
let gridPerfectSquare:boolean;
let rowsInput:number;
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
let backgroundLoaded: boolean;
let logoLoaded: boolean;
let fontsLoaded: boolean;
let showMockup: boolean;
let grid: Grid;

const white = "#fff";
const black = "#000";
const cyan = "#00ffff";
const magenta = "#ff00ff";
const yellow = "#ffff00"
const red = "#ff0000";
const green = "#00ff00";
const blue = "#0000ff";

const baseRows = 15;
const baseCols = 45;
const baseCanvasWidth = 3242;
const baseCanvasHeight = 1080;
const getRowThickness = () => grid.rows / baseRows;
const getColThickness = () => grid.cols / baseCols;

resetAll();

function resetAll() {
  width = 3242;
  height = 1080;
  gridPerfectSquare = false;
  rowsInput = 15;
  showLabel = true;
  labelText = "A1";
  labelLeftColor = "#ffffff";
  labelRightColor = "#000000";
  labelSize = 1.0;
  showBackground = true;
  showLogo = true;
  logoSize = 1.0;
  gridColor = "#ffffff";
  gridThickness = 1.0;
  resizeAndDraw();
}

function drawBackground(canvasWidth:number, canvasHeight:number) {
  if (!ctx) return;
  if (!backgroundLoaded) return;
  if (!showBackground)  return;

  // cover (envelope) the canvas, maintaining aspect ratio
  const canvasRatio = canvasWidth / canvasHeight;
  const imgRatio = backgroundImage.naturalWidth / backgroundImage.naturalHeight;

  let drawWidth, drawHeight, offsetX, offsetY;
  if (imgRatio > canvasRatio) {
    // image wider → crop sides
    drawHeight = canvasHeight;
    drawWidth = canvasHeight * imgRatio;
    offsetX = (canvasWidth - drawWidth) / 2;
    offsetY = 0;
  } else {
    // image taller → crop top/bottom
    drawWidth = canvasWidth;
    drawHeight = canvasWidth / imgRatio;
    offsetX = 0;
    offsetY = (canvasHeight - drawHeight) / 2;
  }

  ctx.drawImage(backgroundImage, offsetX, offsetY, drawWidth, drawHeight);
}

function drawGrid(canvasWidth:number, canvasHeight:number) {
  if (!ctx) return;

  const cellHeight = canvasHeight / rowsInput;
  const rows = rowsInput;
  const cols = Math.floor(canvasWidth / cellHeight);
  const cellWidth = gridPerfectSquare ? cellHeight : canvasWidth / cols;
  const gridWidth = cols * cellWidth;
  const gridHeight = rows * cellHeight;

  const parent = canvasRef.parentElement as HTMLDivElement | null;
  let parentWidth = canvasWidth;
  let parentHeight = canvasHeight;
  if (parent) {
    const renderedCanvasRect = canvasRef.getBoundingClientRect();
    const scaleX = canvasWidth / renderedCanvasRect.width;
    const scaleY = canvasHeight / renderedCanvasRect.height;

    const parentRect = parent.getBoundingClientRect();
    parentWidth = parentRect.width * scaleX;
    parentHeight = parentRect.height * scaleY;
  }

  const startX = (canvasWidth - gridWidth) / 2 + (parentWidth - canvasWidth) / 2;
  const startY = (canvasHeight - gridHeight) / 2 + (parentHeight - canvasHeight) / 2;

  ctx.save();
  ctx.strokeStyle = gridColor;
  ctx.lineWidth = gridThickness * 1.5 * Math.max(Math.ceil(width / baseCanvasWidth), Math.ceil(height / baseCanvasHeight));
  console.log(ctx.lineWidth);

  // draw horizontal lines
  for (let r = 0; r <= rows; r++) {
    const y = startY + r * cellHeight;
    ctx.beginPath();
    ctx.moveTo(0, y);
    ctx.lineTo(canvasWidth, y);
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
  const fontSize = cellHeight * 0.5;
  ctx.font = `bold ${fontSize}px Sora`;
  ctx.fillStyle = gridColor;
  const metrics = ctx.measureText("0");
  const yOffset = metrics.actualBoundingBoxDescent / 2;
  const topRowY = startY + cellHeight / 2 + yOffset;
  const bottomRowY = startY + (rows - 1) * cellHeight + cellHeight / 2 + yOffset;
  const centerCol = Math.floor(cols / 2);

  for (let c = 0; c < cols; c++) {
    const x = startX + c * cellWidth + cellWidth / 2;
    const distFromCenter = Math.abs(c - centerCol);

    let label: number = c;
    // if (cols % 2 === 0) {
    //   const leftCenter = cols / 2 - 1;
    //   const rightCenter = cols / 2;
    //   const mid = (leftCenter + rightCenter) / 2;
    //   label = Math.max(1, Math.round(Math.abs(c - mid)));
    // } else {
    //   // Odd → use actual center column
    //   label = Math.abs(distFromCenter);
    // }

    ctx.fillText(label.toString(), x, topRowY);
    ctx.fillText(label.toString(), x, bottomRowY);
  }

  return { gridWidth, gridHeight, cellWidth, cellHeight, cols, rows, startX, startY } as Grid;
}

function drawLogo() {
  if (!ctx) return;
  if (!logoLoaded) return;
  if (!showLogo) return;

  const { gridWidth, gridHeight, cellHeight, rows, startX, startY } = grid;  
  const centerX = startX + gridWidth / 2;
  const centerY = startY + gridHeight / 2;

  const offset = Math.floor(8 * getRowThickness());
  const maxLogoHeight = (rows - offset) * cellHeight; // leave 2-cell margin top & bottom
  const aspect = logoImage.naturalWidth / logoImage.naturalHeight;
  const imgHeight = Math.min(maxLogoHeight, gridHeight - 4 * cellHeight) * logoSize;
  const imgWidth = imgHeight * aspect;

  ctx.drawImage(
    logoImage,
    centerX - imgWidth / 2,
    centerY - imgHeight / 2,
    imgWidth,
    imgHeight
  );
}

function drawLabelText() {
  if (!ctx) return;
  if (!showLabel) return;

  const { gridHeight, cellWidth, cols, startX } = grid;
  const labelY = gridHeight * .475;
  const fontSize = gridHeight / 2;
  ctx.font = `bold ${fontSize}px Sora`;
  ctx.fillStyle = labelLeftColor;
  const metrics = ctx.measureText("0");
  const yOffset = metrics.actualBoundingBoxDescent / 2;
  const offset = Math.floor(10.5 * getColThickness());
  ctx.fillText(labelText, startX + offset * cellWidth, labelY + yOffset);
  ctx.fillStyle = labelRightColor;
  ctx.fillText(labelText, startX + (cols - offset) * cellWidth, labelY + yOffset);
}

function drawFillCell(x: number, y: number, color = gridColor) {
  if (!ctx || !grid) return;

  const { startX, startY, cellWidth, cellHeight, cols, rows } = grid;

  const clampedX = Math.max(0, Math.min(x, cols - 1));
  const clampedY = Math.max(0, Math.min(y, rows - 1));
  const rowScale = getRowThickness(); 
  const colScale = getColThickness(); 
  const drawX = startX + clampedX * cellWidth;
  const drawY = startY + clampedY * cellHeight;
  const centerX = drawX + cellWidth / 2;
  const centerY = drawY + cellHeight / 2;

  ctx.save();
  ctx.translate(centerX * colScale, centerY * rowScale); 
  ctx.scale(colScale, rowScale);
  ctx.fillStyle = color;
  ctx.fillRect(-cellWidth / 2, -cellHeight / 2, cellWidth, cellHeight);
  ctx.restore();
}

function drawCanvas() {
  if (!ctx) return;
  if (!backgroundLoaded || !logoLoaded || !fontsLoaded) return;

  const { width: canvasWidth, height: canvasHeight } = ctx.canvas;
  ctx.clearRect(0, 0, canvasWidth, canvasHeight);
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";

  drawBackground(canvasWidth, canvasHeight);
  grid = drawGrid(canvasWidth, canvasHeight) as Grid;
  if (grid) {
    drawLabelText();
    drawLogo();
  }

  // logo corner left-top
  drawFillCell(17, 2, magenta);
  drawFillCell(18, 2, magenta);
  drawFillCell(17, 3, yellow);

  // logo corner right-top
  drawFillCell(grid.cols - 19, 2, green);
  drawFillCell(grid.cols - 18, 2, green);
  drawFillCell(grid.cols - 18, 3, cyan)

  // logo corner left-bottom
  drawFillCell(17, 11, yellow);
  drawFillCell(18, 12, blue);
  drawFillCell(17, 12, blue);

  // logo corner right-bottom
  drawFillCell(grid.cols - 18, 11, magenta);
  drawFillCell(grid.cols - 19, 12, cyan);
  drawFillCell(grid.cols - 18, 12, cyan);

  // left
  for (let i = 0; i <= 4; i++)
    drawFillCell(1, 2 + i, red);
  for (let i = 0; i <= 2; i++)
    drawFillCell(2, 2 + i, green);
  for (let i = 4; i >= 0; i--)
    drawFillCell(1, 12 - i, cyan);
  for (let i = 2; i >= 0; i--)
    drawFillCell(2, 12 - i, yellow);
  drawFillCell(1, 7, white);

  drawFillCell(4, 3, white);
  drawFillCell(5, 2, white);
  drawFillCell(6, 2, white);

  drawFillCell(4, 11, black);
  drawFillCell(5, 12, black);
  drawFillCell(6, 12, black);

  // right
  for (let i = 0; i <= 4; i++)
    drawFillCell(grid.cols - 2, 2 + i, cyan);
  for (let i = 0; i <= 2; i++)
    drawFillCell(grid.cols - 3, 2 + i, yellow);
  for (let i = 4; i >= 0; i--)
    drawFillCell(grid.cols - 2, 12 - i, red);
  for (let i = 2; i >= 0; i--)
    drawFillCell(grid.cols - 3, 12 - i, green);
  drawFillCell(grid.cols - 2, 7, white);

  drawFillCell(grid.cols - 5, 3, black);
  drawFillCell(grid.cols - 6, 2, black);
  drawFillCell(grid.cols - 7, 2, black);

  drawFillCell(grid.cols - 5, 11, white);
  drawFillCell(grid.cols - 6, 12, white);
  drawFillCell(grid.cols - 7, 12, white);
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

onMount(async () => {
  ctx = canvasRef.getContext("2d");
  backgroundImage = new Image();
  backgroundImage.src = "./background.png";
  backgroundImage.onload = () => backgroundLoaded = true;

  logoImage = new Image();
  logoImage.src = "./logo.png";
  logoImage.onload = () => logoLoaded = true;

  if (document.fonts && document.fonts.ready) {
    await document.fonts.ready;
    fontsLoaded = true;
  } else {
    console.warn("Font Loading API not supported, skipping wait");
    fontsLoaded = true;
  }

  resizeAndDraw();
});

$: {
  width;
  height;
  resizeAndDraw();
}

// reactive redraw whenever configs change
$: {
  gridPerfectSquare;
  rowsInput;
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
  backgroundLoaded;
  logoLoaded;
  fontsLoaded;
  drawCanvas();
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
        <img src="./testCardReference.webp" style={`position: absolute; width: 100%; left: 50%; top: 50%; transform: translate(-50%, -50%); aspect-ratio: 3242 / 1080; display: ${showMockup ? "block" : "none"}`}/>
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
                <label for="gridRowsSlider">Grid Rows: ({rowsInput})</label>
                <input id="gridRowsSlider" min="4" max="48" type="range" bind:value={rowsInput}>
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
                  <span>Mockup:</span>
                  <label class="switch">
                    <input id="mockupToggle" type="checkbox" bind:checked={showMockup}>
                    <span class="slider-toggle round"></span>
                  </label>
                </label>
              </div>
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
                <label for="labelSizeSlider">Label Size: ({labelSize})</label>
                <input id="labelSizeSlider" min="0.1" max="3" step="0.1" type="range" bind:value={labelSize}>
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
