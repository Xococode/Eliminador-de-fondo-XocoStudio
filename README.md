<!DOCTYPE html>
<html lang="es">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Eliminador de Fondos (Todo en Uno)</title>
  <script src="https://cdn.tailwindcss.com"></script>
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/font-awesome/6.4.0/css/all.min.css">
  <style>
    body { user-select: none; }
    .checkerboard-bg {
      background-image: linear-gradient(45deg, #ccc 25%, transparent 25%),
                        linear-gradient(-45deg, #ccc 25%, transparent 25%),
                        linear-gradient(45deg, transparent 75%, #ccc 75%),
                        linear-gradient(-45deg, transparent 75%, #ccc 75%);
      background-size: 20px 20px;
      background-position: 0 0, 0 10px, 10px -10px, -10px 0px;
    }
    .tool-btn.active-tool { background-color: #d1d5db; }
    .btn-feedback { transition: all 0.2s; }
    .btn-feedback.success { background-color: #22c55e !important; color: white !important; }
    .btn-feedback.error { background-color: #ef4444 !important; color: white !important; }
    .btn-feedback.working { background-color: #eab308 !important; color: white !important; cursor: wait; }
  </style>
</head>
<body class="bg-gray-100 flex flex-col min-h-screen overflow-hidden">

<header class="bg-white shadow-md p-4">
  <h1 class="text-2xl font-bold text-gray-800">Eliminador de Fondos de Imágenes</h1>
</header>

<main class="flex-grow flex flex-col md:flex-row p-4 gap-4">

  <!-- Panel lateral -->
  <aside class="w-full md:w-72 bg-white p-4 rounded-lg shadow space-y-6 overflow-y-auto">
    <!-- Carga de imagen -->
    <div>
      <h2 class="text-lg font-semibold mb-2">1. Cargar Imagen</h2>
      <div id="drop-zone" class="border-2 border-dashed border-gray-400 rounded-lg p-6 text-center cursor-pointer hover:border-blue-500 hover:bg-blue-50 transition">
        <p class="text-gray-600">Arrastra y suelta una imagen</p>
        <p class="text-sm text-gray-500 my-2">o</p>
        <input type="file" id="file-input" accept="image/*" class="hidden">
        <button id="upload-btn" class="w-full bg-blue-500 hover:bg-blue-700 text-white font-bold py-2 px-4 rounded btn-feedback mt-2">
          <i class="fas fa-upload mr-1"></i> Seleccionar Archivo
        </button>
      </div>
    </div>

    <!-- Exportar -->
    <div id="export-section" class="space-y-2 pt-4 border-t">
      <h2 class="text-lg font-semibold mb-2">2. Acciones</h2>
      <button id="export-btn" class="w-full bg-green-600 hover:bg-green-700 text-white font-bold py-2 px-4 rounded btn-feedback" disabled>
        <i class="fas fa-download mr-1"></i> Exportar PNG
      </button>
      <button id="remove-bg-btn" class="w-full bg-red-600 hover:bg-red-700 text-white font-bold py-2 px-4 rounded btn-feedback" disabled>
        <i class="fas fa-magic mr-1"></i> Eliminar Fondo
      </button>
    </div>
  </aside>

  <!-- Área de trabajo -->
  <section class="flex-grow bg-gray-300 rounded-lg shadow overflow-hidden relative checkerboard-bg">
    <div id="canvas-outer-container" class="w-full h-full relative">
      <div id="canvas-inner-container" class="absolute top-0 left-0">
        <canvas id="image-canvas" class="hidden"></canvas>
      </div>
      <p id="canvas-placeholder" class="absolute inset-0 flex items-center justify-center text-gray-500">Carga una imagen para empezar</p>
    </div>
  </section>

</main>

<script>
document.addEventListener('DOMContentLoaded', () => {
  const dropZone = document.getElementById('drop-zone');
  const fileInput = document.getElementById('file-input');
  const uploadBtn = document.getElementById('upload-btn');
  const exportBtn = document.getElementById('export-btn');
  const removeBgBtn = document.getElementById('remove-bg-btn');
  const canvas = document.getElementById('image-canvas');
  const ctx = canvas.getContext('2d');
  const canvasPlaceholder = document.getElementById('canvas-placeholder');
  const canvasOuterContainer = document.getElementById('canvas-outer-container');
  const canvasInnerContainer = document.getElementById('canvas-inner-container');

  let originalImage = null;

  uploadBtn.addEventListener('click', () => fileInput.click());
  fileInput.addEventListener('change', (e) => {
    const file = e.target.files[0];
    if (file && file.type.startsWith('image/')) {
      loadImage(file);
    }
  });

  dropZone.addEventListener('dragover', (e) => {
    e.preventDefault();
    dropZone.classList.add('border-blue-500', 'bg-blue-50');
  });

  dropZone.addEventListener('dragleave', (e) => {
    e.preventDefault();
    dropZone.classList.remove('border-blue-500', 'bg-blue-50');
  });

  dropZone.addEventListener('drop', (e) => {
    e.preventDefault();
    dropZone.classList.remove('border-blue-500', 'bg-blue-50');
    const file = e.dataTransfer.files[0];
    if (file && file.type.startsWith('image/')) {
      loadImage(file);
    }
  });

  exportBtn.addEventListener('click', () => {
    const link = document.createElement('a');
    link.download = 'imagen_sin_fondo.png';
    link.href = canvas.toDataURL('image/png');
    link.click();
  });

  removeBgBtn.addEventListener('click', removeBackground);

  function loadImage(file) {
    const reader = new FileReader();
    reader.onload = (e) => {
      const img = new Image();
      img.onload = () => {
        originalImage = img;
        canvas.width = img.width;
        canvas.height = img.height;
        canvasInnerContainer.style.width = img.width + 'px';
        canvasInnerContainer.style.height = img.height + 'px';
        ctx.clearRect(0, 0, canvas.width, canvas.height);
        ctx.drawImage(img, 0, 0);
        canvasPlaceholder.classList.add('hidden');
        canvas.classList.remove('hidden');
        exportBtn.disabled = false;
        removeBgBtn.disabled = false;
      };
      img.src = e.target.result;
    };
    reader.readAsDataURL(file);
  }

  function removeBackground() {
    if (!originalImage) return;
    showButtonFeedback(removeBgBtn, 'working', 'Eliminando...');
    const imgData = ctx.getImageData(0, 0, canvas.width, canvas.height);
    const data = imgData.data;

    const r0 = data[0], g0 = data[1], b0 = data[2];
    const tolerance = 40; // Puedes ajustar esta tolerancia

    for (let i = 0; i < data.length; i += 4) {
      const r = data[i], g = data[i + 1], b = data[i + 2];
      const distance = Math.sqrt((r - r0) ** 2 + (g - g0) ** 2 + (b - b0) ** 2);
      if (distance < tolerance) {
        data[i + 3] = 0; // Hacer pixel transparente
      }
    }

    ctx.putImageData(imgData, 0, 0);
    showButtonFeedback(removeBgBtn, 'success', '¡Listo!');
  }

  function showButtonFeedback(button, status, message) {
    button.disabled = true;
    const original = button.innerHTML;
    switch (status) {
      case 'working':
        button.innerHTML = `<i class="fas fa-spinner fa-spin mr-1"></i> ${message}`;
        break;
      case 'success':
        button.innerHTML = `<i class="fas fa-check mr-1"></i> ${message}`;
        setTimeout(() => {
          button.innerHTML = original;
          button.disabled = false;
        }, 1500);
        break;
      case 'error':
        button.innerHTML = `<i class="fas fa-times mr-1"></i> ${message}`;
        setTimeout(() => {
          button.innerHTML = original;
          button.disabled = false;
        }, 1500);
        break;
    }
  }

});
</script>

</body>
</html>
