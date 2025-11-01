<template>
  <div class="photobooth-container">
    <!-- Camera View - Fullscreen -->
    <div v-if="!previewMode" class="camera-fullscreen">
      <video ref="videoElement" autoplay playsinline class="video-stream" :class="{ 'flip-horizontal': flipCamera, [`filter-${selectedFilter}`]: selectedFilter !== 'none' }"></video>

      <!-- Countdown Overlay -->
      <div v-if="countdown > 0" class="countdown-overlay">
        <div class="countdown-number">{{ countdown }}</div>
      </div>

      <!-- Loading/Error Overlay -->
      <div v-if="!isStreaming && !error" class="loading-overlay">
        <div class="spinner"></div>
        <p>Starting camera...</p>
      </div>
      <div v-if="error" class="error-overlay">
        <p>{{ error }}</p>
        <button @click="startCamera" class="retry-btn">Try Again</button>
      </div>

      <!-- Header Overlay -->
      <div class="header-overlay">
        <div class="photobooth-header">
          <h1>📸 PhotoBooth</h1>
          <p>Capture your perfect moment</p>
        </div>
      </div>

      <!-- Controls Overlay -->
      <div class="controls-overlay">
        <div class="controls">
          <button @click="capturePhoto" :disabled="!isStreaming || snapshots.length >= 3" class="control-btn primary capture-btn">📸</button>
          <button @click="toggleFilters" :disabled="!isStreaming" class="control-btn secondary filters-btn">🎨</button>
        </div>

        <!-- Filters Panel -->
        <div v-if="showFilters" class="filters-panel">
          <div v-for="filter in filters" :key="filter.name" @click="selectedFilter = filter.name" :class="['filter-option', { active: selectedFilter === filter.name }]">
            <div :class="['filter-preview', `filter-${filter.name}`]"></div>
            <span>{{ filter.label }}</span>
          </div>
        </div>
      </div>

      <!-- Snapshots Panel -->
      <div v-if="snapshots.length > 0" class="snapshots-panel">
        <div class="snapshots-header">
          <h3>Your Takes ({{ snapshots.length }}/3)</h3>
          <button @click="clearSnapshots" class="close-btn">×</button>
        </div>
        <div class="snapshots-grid">
          <div v-for="(snapshot, index) in snapshots" :key="index" class="snapshot-item">
            <div class="snapshot-thumbnail">
              <img :src="snapshot.url" :alt="`Snapshot ${index + 1}`" />
            </div>
            <div class="snapshot-actions">
              <button @click="downloadSnapshot(index)" class="snapshot-btn download-btn" title="Download">
                <span>💾</span>
              </button>
              <button @click="deleteSnapshot(index)" class="snapshot-btn delete-btn" title="Delete">
                <span>🗑️</span>
              </button>
            </div>
          </div>
          <!-- Empty slots for visual consistency -->
          <div v-for="n in 3 - snapshots.length" :key="`empty-${n}`" class="snapshot-item empty-slot">
            <div class="snapshot-thumbnail empty">
              <span>+</span>
            </div>
          </div>
        </div>
      </div>
    </div>

    <!-- Preview Mode -->
    <div v-if="previewMode" class="preview-fullscreen">
      <canvas ref="canvasElement" class="preview-canvas"></canvas>

      <!-- Preview Header Overlay -->
      <div class="header-overlay">
        <div class="photobooth-header">
          <h1>📸 Preview</h1>
        </div>
      </div>

      <!-- Preview Controls Overlay -->
      <div class="controls-overlay">
        <div class="preview-controls">
          <button @click="previewMode = false" class="control-btn secondary">← Back</button>
          <button @click="downloadPhoto" class="control-btn primary">💾 Download</button>
          <button @click="saveToGallery" class="control-btn primary">🖼️ Save</button>
        </div>
      </div>

      <!-- Gallery -->
      <div v-if="photos.length > 0" class="gallery-section">
        <div class="gallery-header">
          <h2>Your Photos ({{ photos.length }})</h2>
          <button @click="clearGallery" class="clear-btn">Clear All</button>
        </div>
        <div class="gallery-grid">
          <div v-for="(photo, index) in photos" :key="index" class="gallery-item" @click="viewPhoto(photo)">
            <img :src="photo.url" :alt="`Photo ${index + 1}`" />
            <button @click.stop="deletePhoto(index)" class="delete-btn">×</button>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup>
import { ref, onMounted, onBeforeUnmount } from 'vue';

const videoElement = ref(null);
const canvasElement = ref(null);
const isStreaming = ref(false);
const previewMode = ref(false);
const error = ref(null);
const countdown = ref(0);
const flipCamera = ref(false);
const showFilters = ref(false);
const selectedFilter = ref('none');
const photos = ref([]);
const snapshots = ref([]);

const stream = ref(null);

const filters = [
  { name: 'none', label: 'Original' },
  { name: 'grayscale', label: 'B&W' },
  { name: 'sepia', label: 'Sepia' },
  { name: 'vintage', label: 'Vintage' },
  { name: 'bright', label: 'Bright' },
  { name: 'contrast', label: 'Contrast' },
];

const startCamera = async () => {
  try {
    error.value = null;
    const constraints = {
      video: {
        facingMode: flipCamera.value ? 'user' : 'environment',
        width: { ideal: 1280 },
        height: { ideal: 720 },
      },
    };

    const mediaStream = await navigator.mediaDevices.getUserMedia(constraints);
    stream.value = mediaStream;

    if (videoElement.value) {
      videoElement.value.srcObject = mediaStream;
      isStreaming.value = true;
    }
  } catch (err) {
    error.value = 'Unable to access camera. Please check permissions.';
    console.error('Camera error:', err);
  }
};

const stopCamera = () => {
  if (stream.value) {
    stream.value.getTracks().forEach((track) => track.stop());
    stream.value = null;
  }
  isStreaming.value = false;
};

const capturePhoto = () => {
  if (!videoElement.value) return;
  if (snapshots.value.length >= 3) return; // Max 3 snapshots

  // Start countdown
  countdown.value = 3;
  const countdownInterval = setInterval(() => {
    countdown.value--;
    if (countdown.value <= 0) {
      clearInterval(countdownInterval);

      // Create a temporary canvas for the snapshot
      const tempCanvas = document.createElement('canvas');
      const video = videoElement.value;

      tempCanvas.width = video.videoWidth;
      tempCanvas.height = video.videoHeight;

      const ctx = tempCanvas.getContext('2d');

      // Draw the video to canvas (flip if needed)
      if (flipCamera.value) {
        ctx.translate(tempCanvas.width, 0);
        ctx.scale(-1, 1);
      }

      ctx.drawImage(video, 0, 0);

      // Always reset transformation to identity before applying filters
      ctx.setTransform(1, 0, 0, 1, 0, 0);

      // Apply filter
      applyFilter(ctx, tempCanvas.width, tempCanvas.height);

      // Save the snapshot
      const snapshotUrl = tempCanvas.toDataURL('image/png');
      snapshots.value.push({
        url: snapshotUrl,
        timestamp: Date.now(),
        canvas: tempCanvas,
      });
    }
  }, 1000);
};

const applyFilter = (ctx, width, height) => {
  // If no filter selected, don't apply anything
  if (selectedFilter.value === 'none') {
    return;
  }

  const imageData = ctx.getImageData(0, 0, width, height);
  const data = imageData.data;

  switch (selectedFilter.value) {
    case 'grayscale':
      for (let i = 0; i < data.length; i += 4) {
        const gray = data[i] * 0.299 + data[i + 1] * 0.587 + data[i + 2] * 0.114;
        data[i] = gray;
        data[i + 1] = gray;
        data[i + 2] = gray;
      }
      break;
    case 'sepia':
      for (let i = 0; i < data.length; i += 4) {
        const r = data[i];
        const g = data[i + 1];
        const b = data[i + 2];
        data[i] = Math.min(255, r * 0.393 + g * 0.769 + b * 0.189);
        data[i + 1] = Math.min(255, r * 0.349 + g * 0.686 + b * 0.168);
        data[i + 2] = Math.min(255, r * 0.272 + g * 0.534 + b * 0.131);
      }
      break;
    case 'vintage':
      for (let i = 0; i < data.length; i += 4) {
        data[i] = Math.min(255, data[i] * 1.1);
        data[i + 1] = Math.min(255, data[i + 1] * 1.05);
        data[i + 2] = Math.max(0, data[i + 2] * 0.9);
      }
      break;
    case 'bright':
      for (let i = 0; i < data.length; i += 4) {
        data[i] = Math.min(255, data[i] * 1.2);
        data[i + 1] = Math.min(255, data[i + 1] * 1.2);
        data[i + 2] = Math.min(255, data[i + 2] * 1.2);
      }
      break;
    case 'contrast':
      const factor = 1.5;
      for (let i = 0; i < data.length; i += 4) {
        data[i] = Math.min(255, Math.max(0, (data[i] - 128) * factor + 128));
        data[i + 1] = Math.min(255, Math.max(0, (data[i + 1] - 128) * factor + 128));
        data[i + 2] = Math.min(255, Math.max(0, (data[i + 2] - 128) * factor + 128));
      }
      break;
    default:
      // If filter not recognized, don't modify
      return;
  }

  ctx.putImageData(imageData, 0, 0);
};

const downloadPhoto = () => {
  if (!canvasElement.value) return;

  canvasElement.value.toBlob((blob) => {
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = `photobooth-${Date.now()}.png`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
    URL.revokeObjectURL(url);
  });
};

const saveToGallery = () => {
  if (!canvasElement.value) return;

  const url = canvasElement.value.toDataURL('image/png');
  photos.value.push({
    url,
    timestamp: Date.now(),
  });

  previewMode.value = false;
};

const viewPhoto = (photo) => {
  if (canvasElement.value) {
    const img = new Image();
    img.onload = () => {
      canvasElement.value.width = img.width;
      canvasElement.value.height = img.height;
      const ctx = canvasElement.value.getContext('2d');
      ctx.drawImage(img, 0, 0);
      previewMode.value = true;
    };
    img.src = photo.url;
  }
};

const deletePhoto = (index) => {
  photos.value.splice(index, 1);
};

const clearGallery = () => {
  if (confirm('Are you sure you want to clear all photos?')) {
    photos.value = [];
  }
};

const downloadSnapshot = (index) => {
  const snapshot = snapshots.value[index];
  if (!snapshot) return;

  // If canvas exists, use it; otherwise create from image URL
  if (snapshot.canvas) {
    snapshot.canvas.toBlob((blob) => {
      const url = URL.createObjectURL(blob);
      const a = document.createElement('a');
      a.href = url;
      a.download = `snapshot-${Date.now()}-${index + 1}.png`;
      document.body.appendChild(a);
      a.click();
      document.body.removeChild(a);
      URL.revokeObjectURL(url);
    });
  } else {
    // Fallback: create download link from data URL
    const a = document.createElement('a');
    a.href = snapshot.url;
    a.download = `snapshot-${Date.now()}-${index + 1}.png`;
    document.body.appendChild(a);
    a.click();
    document.body.removeChild(a);
  }
};

const deleteSnapshot = (index) => {
  snapshots.value.splice(index, 1);
};

const clearSnapshots = () => {
  snapshots.value = [];
};

const toggleFilters = () => {
  showFilters.value = !showFilters.value;
};

const toggleFlipCamera = async () => {
  flipCamera.value = !flipCamera.value;
  stopCamera();
  await new Promise((resolve) => setTimeout(resolve, 100));
  startCamera();
};

onMounted(() => {
  startCamera();
});

onBeforeUnmount(() => {
  stopCamera();
});
</script>

<style scoped>
.photobooth-container {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #000;
}

/* Camera Fullscreen */
.camera-fullscreen {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
}

.video-stream {
  position: absolute;
  top: 0;
  left: 0;
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
  transition: filter 0.3s ease;
}

.flip-horizontal {
  transform: scaleX(-1);
}

/* Live Preview Filters */
.video-stream.filter-grayscale {
  filter: grayscale(100%);
}

.video-stream.filter-sepia {
  filter: sepia(100%);
}

.video-stream.filter-vintage {
  filter: brightness(1.1) saturate(1.2) contrast(1.1);
}

.video-stream.filter-bright {
  filter: brightness(1.2);
}

.video-stream.filter-contrast {
  filter: contrast(1.5);
}

/* Header Overlay */
.header-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  padding: 20px;
  background: linear-gradient(180deg, rgba(0, 0, 0, 0.7) 0%, rgba(0, 0, 0, 0.4) 50%, transparent 100%);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  z-index: 20;
  pointer-events: none;
}

.photobooth-header {
  text-align: center;
  color: white;
  pointer-events: auto;
}

.photobooth-header h1 {
  font-size: clamp(1.5rem, 4vw, 2.5rem);
  margin-bottom: 5px;
  text-shadow: 2px 2px 8px rgba(0, 0, 0, 0.8);
  font-weight: 600;
}

.photobooth-header p {
  font-size: clamp(0.9rem, 2vw, 1.1rem);
  opacity: 0.95;
  text-shadow: 1px 1px 4px rgba(0, 0, 0, 0.8);
}

/* Controls Overlay */
.controls-overlay {
  position: absolute;
  bottom: 0;
  left: 0;
  right: 0;
  padding: 20px;
  background: transparent;
  z-index: 20;
}

.countdown-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.7);
  display: flex;
  align-items: center;
  justify-content: center;
  z-index: 30;
}

.countdown-number {
  font-size: clamp(6rem, 15vw, 12rem);
  color: white;
  font-weight: bold;
  animation: pulse 1s ease-in-out;
  text-shadow: 0 0 30px rgba(255, 255, 255, 0.8);
}

@keyframes pulse {
  0%,
  100% {
    transform: scale(1);
  }
  50% {
    transform: scale(1.2);
  }
}

.loading-overlay,
.error-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0, 0, 0, 0.9);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  color: white;
  z-index: 30;
}

.spinner {
  width: 50px;
  height: 50px;
  border: 4px solid rgba(255, 255, 255, 0.3);
  border-top: 4px solid white;
  border-radius: 50%;
  animation: spin 1s linear infinite;
  margin-bottom: 20px;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }
  100% {
    transform: rotate(360deg);
  }
}

.retry-btn {
  margin-top: 20px;
  padding: 10px 20px;
  background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 1rem;
  transition: all 0.3s;
  box-shadow: 0 4px 15px rgba(102, 126, 234, 0.3);
}

.retry-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 6px 20px rgba(102, 126, 234, 0.4);
}

.controls {
  display: flex;
  gap: 20px;
  justify-content: center;
  align-items: center;
  flex-wrap: wrap;
  margin-bottom: 10px;
  position: relative;
  width: 100%;
}

.control-btn {
  padding: 0;
  border: none;
  border-radius: 50%;
  font-size: 1.5rem;
  font-weight: 500;
  cursor: pointer;
  transition: all 0.3s cubic-bezier(0.4, 0, 0.2, 1);
  width: 70px;
  height: 70px;
  display: flex;
  align-items: center;
  justify-content: center;
  position: relative;
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  background: rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.2);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3), inset 0 1px 1px rgba(255, 255, 255, 0.2), inset 0 -1px 1px rgba(0, 0, 0, 0.1);
  overflow: hidden;
  -webkit-tap-highlight-color: transparent;
}

.control-btn::before {
  content: '';
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  height: 50%;
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.15) 0%, rgba(255, 255, 255, 0) 100%);
  border-radius: 50% 50% 0 0;
  pointer-events: none;
}

.control-btn.primary {
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.3) 0%, rgba(118, 75, 162, 0.3) 100%);
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow: 0 8px 32px rgba(102, 126, 234, 0.4), inset 0 1px 1px rgba(255, 255, 255, 0.3), inset 0 -1px 1px rgba(0, 0, 0, 0.1);
  color: white;
}

.control-btn.primary::before {
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.25) 0%, rgba(255, 255, 255, 0) 100%);
}

.control-btn.primary:hover:not(:disabled) {
  transform: translateY(-3px) scale(1.05);
  background: linear-gradient(135deg, rgba(102, 126, 234, 0.4) 0%, rgba(118, 75, 162, 0.4) 100%);
  border-color: rgba(255, 255, 255, 0.4);
  box-shadow: 0 12px 40px rgba(102, 126, 234, 0.5), inset 0 1px 1px rgba(255, 255, 255, 0.35), inset 0 -1px 1px rgba(0, 0, 0, 0.1);
}

.control-btn.secondary {
  background: rgba(255, 255, 255, 0.08);
  border: 1px solid rgba(255, 255, 255, 0.15);
  color: rgba(255, 255, 255, 0.95);
  box-shadow: 0 8px 32px rgba(0, 0, 0, 0.3), inset 0 1px 1px rgba(255, 255, 255, 0.15), inset 0 -1px 1px rgba(0, 0, 0, 0.1);
}

.control-btn.secondary:hover:not(:disabled) {
  transform: translateY(-3px) scale(1.05);
  background: rgba(255, 255, 255, 0.12);
  border-color: rgba(255, 255, 255, 0.25);
  box-shadow: 0 12px 40px rgba(0, 0, 0, 0.4), inset 0 1px 1px rgba(255, 255, 255, 0.2), inset 0 -1px 1px rgba(0, 0, 0, 0.1);
}

.control-btn:active:not(:disabled) {
  transform: translateY(-1px) scale(0.98);
}

.control-btn:disabled {
  opacity: 0.4;
  cursor: not-allowed;
  transform: none;
}

.capture-btn {
  background: linear-gradient(135deg, rgba(240, 147, 251, 0.35) 0%, rgba(245, 87, 108, 0.35) 100%);
  border: 1px solid rgba(255, 255, 255, 0.3);
  box-shadow: 0 8px 32px rgba(245, 87, 108, 0.5), inset 0 1px 1px rgba(255, 255, 255, 0.3), inset 0 -1px 1px rgba(0, 0, 0, 0.1);
  width: 90px;
  height: 90px;
  display: flex;
  align-items: center;
  justify-content: center;
  padding-bottom: 14px;
}

.capture-btn::before {
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.3) 0%, rgba(255, 255, 255, 0) 100%);
}

.capture-btn:hover:not(:disabled) {
  background: linear-gradient(135deg, rgba(240, 147, 251, 0.45) 0%, rgba(245, 87, 108, 0.45) 100%);
  border-color: rgba(255, 255, 255, 0.4);
  box-shadow: 0 12px 40px rgba(245, 87, 108, 0.6), inset 0 1px 1px rgba(255, 255, 255, 0.35), inset 0 -1px 1px rgba(0, 0, 0, 0.1);
}

.filters-btn {
  width: 50px;
  height: 50px;
  font-size: 1.2rem;
  position: absolute;
  right: 20px;
  bottom: 0;
}

.filters-btn::before {
  background: linear-gradient(180deg, rgba(255, 255, 255, 0.15) 0%, rgba(255, 255, 255, 0) 100%);
}

.filters-panel {
  display: flex;
  gap: 12px;
  justify-content: center;
  flex-wrap: wrap;
  padding: 15px 20px;
  background: rgba(37, 37, 56, 0.9);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  border-radius: 20px;
  margin-top: 15px;
  border: 1px solid rgba(255, 255, 255, 0.15);
  box-shadow: 0 4px 20px rgba(0, 0, 0, 0.3);
}

.filter-option {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 8px;
  cursor: pointer;
  padding: 10px;
  border-radius: 10px;
  transition: all 0.3s;
  color: #e0e0e0;
}

.filter-option:hover {
  background: #3d3d54;
  transform: scale(1.05);
}

.filter-option.active {
  background: #667eea;
  color: white;
}

.filter-preview {
  width: 60px;
  height: 60px;
  border-radius: 50%;
  border: 3px solid transparent;
  background: linear-gradient(45deg, #ff6b6b, #4ecdc4);
}

.filter-preview.filter-grayscale {
  filter: grayscale(100%);
}

.filter-preview.filter-sepia {
  filter: sepia(100%);
}

.filter-preview.filter-vintage {
  filter: brightness(1.1) saturate(1.2) contrast(1.1);
}

.filter-preview.filter-bright {
  filter: brightness(1.2);
}

.filter-preview.filter-contrast {
  filter: contrast(1.5);
}

/* Snapshots Panel */
.snapshots-panel {
  position: fixed;
  top: 10px;
  left: 50%;
  transform: translateX(-50%);
  width: min(90%, 600px);
  max-height: 400px;
  background: rgba(30, 30, 46, 0.95);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 24px;
  padding: 20px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.5), inset 0 1px 1px rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.15);
  z-index: 30;
  animation: slideUp 0.3s ease-out;
}

@keyframes slideUp {
  from {
    opacity: 0;
    transform: translateX(-50%) translateY(20px);
  }
  to {
    opacity: 1;
    transform: translateX(-50%) translateY(0);
  }
}

.snapshots-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 16px;
}

.snapshots-header h3 {
  color: rgba(255, 255, 255, 0.95);
  font-size: 1.2rem;
  font-weight: 600;
  margin: 0;
  text-shadow: 0 1px 3px rgba(0, 0, 0, 0.3);
}

.close-btn {
  width: 32px;
  height: 32px;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(255, 255, 255, 0.1);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  color: rgba(255, 255, 255, 0.9);
  font-size: 1.5rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s;
  line-height: 1;
  -webkit-tap-highlight-color: transparent;
}

.close-btn:hover {
  background: rgba(255, 255, 255, 0.15);
  border-color: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

.snapshots-grid {
  display: grid;
  grid-template-columns: repeat(3, 1fr);
  gap: 12px;
}

.snapshot-item {
  position: relative;
  border-radius: 16px;
  overflow: hidden;
  aspect-ratio: 9 / 16;
  background: rgba(0, 0, 0, 0.3);
  border: 1px solid rgba(255, 255, 255, 0.1);
  transition: transform 0.3s;
}

.snapshot-item:hover {
  transform: scale(1.02);
}

.snapshot-thumbnail {
  width: 100%;
  height: 100%;
  position: relative;
  overflow: hidden;
}

.snapshot-thumbnail img {
  width: 100%;
  height: 100%;
  object-fit: cover;
  display: block;
}

.snapshot-thumbnail.empty {
  display: flex;
  align-items: center;
  justify-content: center;
  background: rgba(0, 0, 0, 0.2);
  border: 2px dashed rgba(255, 255, 255, 0.2);
}

.snapshot-thumbnail.empty span {
  font-size: 2rem;
  color: rgba(255, 255, 255, 0.3);
}

.snapshot-actions {
  position: absolute;
  bottom: 8px;
  left: 50%;
  transform: translateX(-50%);
  display: flex;
  gap: 8px;
  opacity: 0;
  transition: opacity 0.3s;
}

.snapshot-item:hover .snapshot-actions {
  opacity: 1;
}

.snapshot-btn {
  width: 36px;
  height: 36px;
  border-radius: 50%;
  border: 1px solid rgba(255, 255, 255, 0.2);
  background: rgba(0, 0, 0, 0.6);
  backdrop-filter: blur(10px);
  -webkit-backdrop-filter: blur(10px);
  color: white;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: all 0.3s;
  font-size: 1rem;
  -webkit-tap-highlight-color: transparent;
}

.snapshot-btn:hover {
  background: rgba(0, 0, 0, 0.8);
  border-color: rgba(255, 255, 255, 0.3);
  transform: scale(1.1);
}

.snapshot-btn.download-btn:hover {
  background: rgba(102, 126, 234, 0.8);
  border-color: rgba(102, 126, 234, 0.5);
}

.snapshot-btn.delete-btn:hover {
  background: rgba(245, 87, 108, 0.8);
  border-color: rgba(245, 87, 108, 0.5);
}

.snapshot-btn span {
  display: block;
  line-height: 1;
}

/* Preview Fullscreen */
.preview-fullscreen {
  position: relative;
  width: 100vw;
  height: 100vh;
  overflow: hidden;
  background: #000;
  display: flex;
  align-items: center;
  justify-content: center;
}

.preview-canvas {
  width: 100%;
  height: 100%;
  object-fit: contain;
  display: block;
}

.preview-controls {
  display: flex;
  gap: 15px;
  justify-content: center;
  flex-wrap: wrap;
  margin-bottom: 10px;
}

.gallery-section {
  position: absolute;
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
  width: min(90%, 800px);
  max-height: 70vh;
  overflow-y: auto;
  background: rgba(30, 30, 46, 0.95);
  backdrop-filter: blur(20px);
  -webkit-backdrop-filter: blur(20px);
  border-radius: 24px;
  padding: 20px;
  box-shadow: 0 20px 60px rgba(0, 0, 0, 0.7), inset 0 1px 1px rgba(255, 255, 255, 0.1);
  border: 1px solid rgba(255, 255, 255, 0.15);
  z-index: 25;
}

.gallery-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  margin-bottom: 20px;
}

.gallery-header h2 {
  color: #e0e0e0;
}

.clear-btn {
  padding: 10px 20px;
  background: #f5576c;
  color: white;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-size: 0.9rem;
  transition: background 0.3s;
}

.clear-btn:hover {
  background: #e44559;
}

.gallery-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 20px;
}

.gallery-item {
  position: relative;
  border-radius: 15px;
  overflow: hidden;
  aspect-ratio: 1;
  cursor: pointer;
  transition: transform 0.3s;
  box-shadow: 0 4px 15px rgba(0, 0, 0, 0.1);
}

.gallery-item:hover {
  transform: scale(1.05);
}

.gallery-item img {
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.delete-btn {
  position: absolute;
  top: 10px;
  right: 10px;
  width: 30px;
  height: 30px;
  border-radius: 50%;
  background: rgba(255, 0, 0, 0.8);
  color: white;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  display: flex;
  align-items: center;
  justify-content: center;
  transition: background 0.3s;
}

.delete-btn:hover {
  background: rgba(255, 0, 0, 1);
}

@media (max-width: 768px) {
  .header-overlay {
    padding: 15px;
  }

  .controls-overlay {
    padding: 15px;
  }

  .controls {
    gap: 10px;
  }

  .control-btn {
    width: 60px;
    height: 60px;
    font-size: 1.3rem;
  }

  .capture-btn {
    width: 80px;
    height: 80px;
    font-size: 2.5rem;
  }

  .filters-btn {
    width: 45px;
    height: 45px;
    font-size: 1.1rem;
    right: 15px;
  }

  .filters-panel {
    gap: 8px;
    padding: 12px 15px;
  }

  .snapshots-panel {
    width: min(95%, 100%);
    bottom: 100px;
    padding: 15px;
    max-height: 300px;
  }

  .snapshots-grid {
    gap: 8px;
  }

  .snapshot-item {
    border-radius: 12px;
  }

  .snapshot-btn {
    width: 32px;
    height: 32px;
    font-size: 0.9rem;
  }

  .snapshots-header h3 {
    font-size: 1rem;
  }

  .gallery-grid {
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
  }
}
</style>
