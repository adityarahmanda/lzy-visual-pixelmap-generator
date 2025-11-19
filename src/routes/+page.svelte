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

interface CanvasImage {
  id?: string;
  image: HTMLImageElement;
  loaded?: boolean;
  fileImage?: File;
}

let canvasRef: HTMLCanvasElement;
let backgroundInput: HTMLInputElement;
let background:CanvasImage;
let logoInput: HTMLInputElement;
let logo: CanvasImage;
let colorBarLeft: CanvasImage;
let colorBarRight: CanvasImage;
let colorBarCenter: CanvasImage;
let ctx: CanvasRenderingContext2D | null = null;

// ---- CONFIG ----
let width:number;
let height:number;
let resizeTimer: any = null;
let gridPerfectSquare:boolean;
let rowsInput:number;
let showLabel:boolean;
let labelText:string;
let labelLeftColor:string;
let labelRightColor:string;
let labelSize:number;
let labelSpaceFromCenter:number;
let showBackground:boolean;
let showLogo:boolean;
let logoSize:number;
let showCenterColorBar:boolean;
let centerColorBarSize:number;
let showLeftRightColorBar:boolean;
let gridColor:string;
let gridThickness:number;
let fontsLoaded: boolean;
let grid: Grid;

const backgroundImagePath = "./Background.webp";
const logoImagePath = "./Logo.webp";
const colorBarLeftImagePath = "./ColorBarLeft.webp";
const colorBarRightImagePath = "./ColorBarRight.webp";
const colorBarCenterImagePath = "./ColorBarCenter.webp";

const defaultImagePathMap = new Map<string, string>([
  ["background", backgroundImagePath],
  ["logo", logoImagePath],
]);

const isLandscape = () => width >= height;
const getBaseRows = () => isLandscape() ? 15 : 45;
const getBaseCols = () => isLandscape() ? 45 : 15;
const getBaseCanvasWidth = () => isLandscape() ? 3242 : 1080;
const getBaseCanvasHeight = () => isLandscape() ? 1080 : 3242;
const getRowThickness = () => grid.rows / getBaseRows();
const getColThickness = () => grid.cols / getBaseCols();

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
  labelSpaceFromCenter = 0.6;
  showBackground = true;
  showLogo = true;
  logoSize = 1.0;
  gridColor = "#ffffff";
  gridThickness = 1.0;
  showCenterColorBar = true;
  centerColorBarSize = 1.0;
  showLeftRightColorBar = true;
  drawCanvas();
}

function drawBackground(canvasWidth:number, canvasHeight:number) {
  if (!ctx) return;
  if (!background || !background.loaded) return;
  if (!showBackground)  return;

  // cover (envelope) the canvas, maintaining aspect ratio
  const canvasRatio = canvasWidth / canvasHeight;
  const imgRatio = background.image.naturalWidth / background.image.naturalHeight;

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

  ctx.drawImage(background.image, offsetX, offsetY, drawWidth, drawHeight);
}

function drawGrid(canvasWidth:number, canvasHeight:number) {
  if (!ctx) return;

  const rows = rowsInput;
  const cellHeight = canvasHeight / rows;
  const cols = Math.floor(canvasWidth / cellHeight);
  const cellWidth = gridPerfectSquare ? cellHeight : canvasWidth / cols;
  const gridWidth = cols * cellWidth;
  const gridHeight = rows * cellHeight;
  const startX = (canvasWidth - gridWidth) / 2;
  const startY = (canvasHeight - gridHeight) / 2;

  ctx.save();
  ctx.strokeStyle = gridColor;
  ctx.lineWidth = gridThickness * 1.5 * Math.max(Math.ceil(width / getBaseCanvasWidth()), Math.ceil(height / getBaseCanvasHeight()));

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
    if (cols % 2 === 0) {
      const leftCenter = cols / 2 - 1;
      const rightCenter = cols / 2;
      const mid = (leftCenter + rightCenter) / 2;
      label = Math.max(1, Math.round(Math.abs(c - mid)));
    } else {
      label = Math.abs(distFromCenter);
    }

    ctx.fillText(label.toString(), x, topRowY);
    ctx.fillText(label.toString(), x, bottomRowY);
  }

  return { gridWidth, gridHeight, cellWidth, cellHeight, cols, rows, startX, startY } as Grid;
}

function drawLogo() {
  if (!ctx) return;
  if (!logo || !logo.loaded) return;
  if (!showLogo) return;

  const { cellWidth, cellHeight, cols, rows, startX, startY } = grid;  
  const centerX = startX + width / 2;
  const centerY = startY + height / 2;

  const offset = Math.floor((isLandscape() ? getRowThickness() : getColThickness()) * 8);
  const aspect = logo.image.naturalWidth / logo.image.naturalHeight;
  let imgWidth, imgHeight;
  if (isLandscape()) {
    imgWidth = (rows - offset) * cellHeight * logoSize;
    imgHeight = imgWidth / aspect;
  } else {
    imgHeight = (cols - offset) * cellWidth * logoSize;
    imgWidth = imgHeight * aspect;
  }

  ctx.drawImage(
    logo.image,
    centerX - imgWidth / 2,
    centerY - imgHeight / 2,
    imgWidth,
    imgHeight
  );
}

function drawLabelText() {
  if (!ctx) return;
  if (!showLabel) return;

  const { cellWidth, cellHeight, gridWidth, gridHeight, cols, rows, startX, startY } = grid;

  if (isLandscape())
  {
    const rowOffset = Math.floor(getRowThickness() * 8);
    const fontSize = (rows - rowOffset) * cellHeight * labelSize;
    ctx.font = `bold ${fontSize}px Sora`;
    const metrics = ctx.measureText(labelText);
    const textYOffset = metrics.actualBoundingBoxDescent / 2;
    ctx.fillStyle = labelLeftColor;
    ctx.fillText(labelText, startX + (gridWidth / 2) * (1 - labelSpaceFromCenter), height / 2 + textYOffset);
    ctx.fillStyle = labelRightColor;
    ctx.fillText(labelText, startX + (gridWidth / 2) + (gridWidth / 2) * labelSpaceFromCenter, height / 2+ textYOffset);
  }
  else
  {
    const colOffset = Math.floor(getColThickness() * 8);
    const fontSize = (cols - colOffset) * cellWidth * labelSize;
    ctx.font = `bold ${fontSize}px Sora`;
    const metrics = ctx.measureText(labelText);
    const textYOffset = metrics.actualBoundingBoxDescent / 2;
    ctx.fillStyle = labelLeftColor;
    ctx.fillText(labelText, width / 2, startY + (gridHeight / 2) * (1 - labelSpaceFromCenter) * cellHeight + textYOffset);
    ctx.fillStyle = labelRightColor;
    ctx.fillText(labelText, width / 2, startY + (gridHeight / 2) + (gridHeight / 2) * labelSpaceFromCenter * cellHeight + textYOffset);
  }
}

function drawColorBarLeftRight() {
  if (!ctx) return;
  if (!colorBarLeft || !colorBarLeft.loaded) return;
  if (!colorBarRight || !colorBarRight.loaded) return;
  if (!showLeftRightColorBar) return;

  const { gridHeight, cellWidth, cellHeight, cols, rows, startX, startY } = grid;
  if (cols < getBaseCols()) return

  const offsetX = Math.floor(getColThickness());
  const offsetY = Math.floor(getRowThickness() * 4);

  // both left and right image should have same size, only mirrored
  const aspect = colorBarLeft.image.naturalWidth / colorBarLeft.image.naturalHeight;
  const imgHeight = (rows - offsetY) * cellHeight;
  const imgWidth = imgHeight * aspect;

  const leftCenterX = startX + offsetX * cellWidth;
  const centerY = startY + gridHeight / 2;
  ctx.drawImage(
    colorBarLeft.image,
    leftCenterX,
    centerY - imgHeight / 2,
    imgWidth,
    imgHeight
  );
  
  const rightCenterX = startX + (cols - offsetX) * cellWidth;
  ctx.drawImage(
    colorBarRight.image,
    rightCenterX - imgWidth,
    centerY - imgHeight / 2,
    imgWidth,
    imgHeight
  );
}

function drawColorBarCenter() {
  if (!ctx) return;
  if (!colorBarCenter || !colorBarCenter.loaded) return;
  if (!showCenterColorBar) return;
  
  const { cellWidth, cellHeight, rows, cols } = grid;
  const offset = Math.floor((isLandscape() ? getRowThickness() : getColThickness()) * 4);
  const aspect = colorBarCenter.image.naturalWidth / colorBarCenter.image.naturalHeight;
  let imgWidth, imgHeight;
  if (isLandscape()) {
    imgWidth = (rows - offset) * cellHeight * centerColorBarSize;
    imgHeight = imgWidth / aspect;
  } else {
    imgHeight = (cols - offset) * cellWidth * centerColorBarSize;
    imgWidth = imgHeight * aspect;
  }

  ctx.drawImage(
    colorBarCenter.image,
    width / 2 - imgWidth / 2,
    height / 2 - imgHeight / 2,
    imgWidth,
    imgHeight
  );
}

function drawCanvas() {
  if (!ctx) return;
  if (width <= 0 || height <= 0) return;

  const { width: canvasWidth, height: canvasHeight } = ctx.canvas;
  ctx.clearRect(0, 0, canvasWidth, canvasHeight);
  ctx.save();
  ctx.textAlign = "center";
  ctx.textBaseline = "middle";

  drawBackground(canvasWidth, canvasHeight);
  grid = drawGrid(canvasWidth, canvasHeight) as Grid;
  if (grid) {
    drawLabelText();
    drawLogo();
  }

  // drawColorBarLeftRight();
  drawColorBarCenter();
  ctx.restore();
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

  background = { 
    id: "background",
    image: new Image(), 
  }
  background.image = new Image();
  background.image.src = backgroundImagePath;
  background.image.onload = () => background.loaded = true;

  logo = { 
    id: "logo",
    image: new Image(), 
  }
  logo.image = new Image();
  logo.image.src = logoImagePath;
  logo.image.onload = () => logo.loaded = true;

  colorBarLeft = { 
    image: new Image(), 
  }
  colorBarLeft.image = new Image();
  colorBarLeft.image.src = colorBarLeftImagePath;
  colorBarLeft.image.onload = () => colorBarLeft.loaded = true;

  colorBarRight = { 
    image: new Image(), 
  }
  colorBarRight.image = new Image();
  colorBarRight.image.src = colorBarRightImagePath;
  colorBarRight.image.onload = () => colorBarRight.loaded = true;

  colorBarCenter = { 
    image: new Image(), 
  }
  colorBarCenter.image = new Image();
  colorBarCenter.image.src = colorBarCenterImagePath;
  colorBarCenter.image.onload = () => colorBarCenter.loaded = true;

  if (document.fonts && document.fonts.ready) {
    await document.fonts.ready;
    fontsLoaded = true;
  } else {
    console.warn("Font Loading API not supported, skipping wait");
    fontsLoaded = true;
  }

  drawCanvas();
});

$: if (fontsLoaded && background?.loaded && logo?.loaded && colorBarCenter?.loaded) {
  width;
  height;
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(() => {
    drawCanvas();
  }, 50);
}

$: if (fontsLoaded && background?.loaded && logo?.loaded && colorBarCenter?.loaded) {
  gridPerfectSquare;
  rowsInput;
  showLabel;
  labelText;
  labelLeftColor;
  labelRightColor;
  labelSize;
  labelSpaceFromCenter;
  showBackground;
  showLogo;
  logoSize;
  gridColor;
  gridThickness;
  showCenterColorBar;
  centerColorBarSize;
  showLeftRightColorBar;
  drawCanvas();
}

function loadCanvasImageFromFile(canvasImage: CanvasImage, file?: File) {
  const canvasImageId = canvasImage.id as string;

  // Determine the source
  let src: string | null = null;

  if (file) {
    src = URL.createObjectURL(file);
    canvasImage.fileImage = file;
  } else if (defaultImagePathMap.has(canvasImageId)) {
    src = defaultImagePathMap.get(canvasImageId)!;
    canvasImage.fileImage = undefined;
  } else {
    return; // nothing to load
  }

  // Begin loading
  canvasImage.loaded = false;
  const img = new Image();
  canvasImage.image = img;

  img.onload = () => {
    canvasImage.loaded = true;
    if (file) URL.revokeObjectURL(src!);
    drawCanvas();
  };

  img.src = src!;
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
              <div class="control-item"><hr/></div>
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
                <label>Custom Background:</label>
                <div class="control-item-row">
                  <div class="file-container">
                    <div class="file-input">
                      <input 
                        bind:this={backgroundInput}
                        type="file" 
                        accept="image/*" 
                        on:change={e => loadCanvasImageFromFile(background, event.target.files[0])}
                      />
                    </div>
                    <div class="file-input-clear">
                      <button class="action-button" on:click={() =>  {
                          loadCanvasImageFromFile(background, null); 
                          backgroundInput.value = "";
                        }}>✖</button>
                    </div>
                  </div>
                </div>
              </div>
              <div class="control-item"><hr/></div>
              <div class="control-item">
                <label for="gridRowsSlider">Grid Rows: ({rowsInput})</label>
                <input id="gridRowsSlider" min="4" max="48" step="1" type="range" bind:value={rowsInput}>
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
              <!-- <div class="control-item">
                <label class="toggle-switch-label">
                  <span>Left-Right Color Bar:</span>
                  <label class="switch">
                    <input id="leftRightColorBarToggle" type="checkbox" bind:checked={showLeftRightColorBar}>
                    <span class="slider-toggle round"></span>
                  </label>
                </label>
              </div> -->
            </div>
            <div class="control-column">
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
                <label>Custom Logo:</label>
                <div class="control-item-row">
                  <div class="file-container">
                    <div class="file-input">
                      <input 
                        bind:this={logoInput}
                        type="file" 
                        accept="image/*" 
                        on:change={e => loadCanvasImageFromFile(logo, event.target.files[0])}
                      />
                    </div>
                    <div class="file-input-clear">
                      <button class="action-button" on:click={() => {
                        loadCanvasImageFromFile(logo, null);
                        logoInput.value = "";
                      }}>✖</button>
                    </div>
                  </div>
                </div>
              </div>
              <div class="control-item">
                <label for="logoSizeSlider">Logo Size: ({logoSize})</label>
                <input id="logoSizeSlider" min="0.1" max="3" step="0.1" type="range" bind:value={logoSize}>
              </div>
              <div class="control-item">
                <label class="toggle-switch-label">
                  <span>Logo Color Bar:</span>
                  <label class="switch">
                    <input id="logoColorBarToggle" type="checkbox" bind:checked={showCenterColorBar}>
                    <span class="slider-toggle round"></span>
                  </label>
                </label>
              </div>
              <div class="control-item">
                <label for="logoColorBarSlider">Logo Color Bar Size: ({centerColorBarSize})</label>
                <input id="logoColorBarSlider" min="0.1" max="3" step="0.1" type="range" bind:value={centerColorBarSize}>
              </div>
              <div class="control-item"><hr/></div>
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
                <label for="labelSpaceFromCenterSlider">Label Space From Center: ({labelSpaceFromCenter})</label>
                <input id="labelSpaceFromCenterSlider" min="0" max="1" step="0.1" type="range" bind:value={labelSpaceFromCenter}>
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
