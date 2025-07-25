<script setup>
import { ref, watch, onMounted } from 'vue';

// Reactive state
const file = ref(null);
const threshold = ref(0.5);
const imageUrl = ref('');
const results = ref([]);
const isLoading = ref(false);
const error = ref('');
const originalImage = ref(null);
const imagePreview = ref(null);

// Service domain state
const serviceDomain = ref(''); // Currently selected/entered domain
const domainList = ref([]); // List of saved domains for the dropdown

// Load domains and last-used domain from localStorage on component mount
onMounted(() => {
  const savedDomains = JSON.parse(localStorage.getItem('domainList') || '[]');
  const lastUsedDomain = localStorage.getItem('serviceDomain') || 'https://ax-img-clip.dev.yiyiny.com';

  if (savedDomains.length > 0) {
    domainList.value = savedDomains;
  } else {
    // Initialize with a default if the list is empty
    domainList.value = [lastUsedDomain];
  }
  
  // Set the current domain to the last one used
  serviceDomain.value = lastUsedDomain;
});

// Watch for changes in the selected domain and save it as the last-used
watch(serviceDomain, (newDomain) => {
  if (newDomain) {
    localStorage.setItem('serviceDomain', newDomain);
  }
});

// Function to add the current domain to the list if it's new
const addDomainToList = () => {
  if (serviceDomain.value && !domainList.value.includes(serviceDomain.value)) {
    domainList.value.push(serviceDomain.value);
    localStorage.setItem('domainList', JSON.stringify(domainList.value));
  }
};

// Trigger detection automatically when a file is selected
const handleFileChange = (event) => {
  const selectedFile = event.target.files[0];
  if (!selectedFile) return;

  file.value = selectedFile;
  imageUrl.value = URL.createObjectURL(selectedFile);
  results.value = []; // Clear previous results
  error.value = '';
};

// Function to run the detection
const runDetection = async () => {
  if (!file.value) {
    return;
  }

  isLoading.value = true;
  error.value = '';
  results.value = [];

  // Add the domain to the list before making the request
  addDomainToList();

  const formData = new FormData();
  formData.append('file', file.value);

  try {
    if (!serviceDomain.value) {
      throw new Error('服务域名不能为空。');
    }
    let url = `${serviceDomain.value}/crop`;
    const response = await fetch(`${url}?threshold=${threshold.value}`, {
      method: 'POST',
      body: formData,
    });

    if (!response.ok) {
      const errorText = await response.text();
      throw new Error(`API 错误: ${response.status} ${response.statusText} - ${errorText}`);
    }

    const data = await response.json();
    results.value = data[0];
    if (data.length === 0) {
      error.value = '未检测到目标。请尝试调整置信度阈值。';
    }
  } catch (e) {
    error.value = e.message || '发生未知错误。';
  } finally {
    isLoading.value = false;
  }
};

// Watch for changes to the image URL (new file uploaded) or threshold, and run detection
watch([imageUrl, threshold], () => {
  if (imageUrl.value) {
    runDetection();
  }
}, { immediate: true });


// Store the original image object once it's loaded
const onImageLoad = (event) => {
  originalImage.value = event.target;
};

// Calculate bbox style based on displayed image dimensions
const getBboxStyle = (bbox) => {
  if (!imagePreview.value || !originalImage.value) return {};

  const scaleX = imagePreview.value.clientWidth / originalImage.value.naturalWidth;
  const scaleY = imagePreview.value.clientHeight / originalImage.value.naturalHeight;

  const [x1, y1, x2, y2] = bbox;
  return {
    left: `${x1 * scaleX}px`,
    top: `${y1 * scaleY}px`,
    width: `${(x2 - x1) * scaleX}px`,
    height: `${(y2 - y1) * scaleY}px`,
  };
};

// Download the cropped area
const downloadCrop = (bbox, index) => {
  if (!originalImage.value) return;

  const [x1, y1, x2, y2] = bbox;
  const cropWidth = x2 - x1;
  const cropHeight = y2 - y1;

  const canvas = document.createElement('canvas');
  canvas.width = cropWidth;
  canvas.height = cropHeight;
  const ctx = canvas.getContext('2d');

  ctx.drawImage(originalImage.value, x1, y1, cropWidth, cropHeight, 0, 0, cropWidth, cropHeight);

  const link = document.createElement('a');
  link.href = canvas.toDataURL('image/png');
  link.download = `裁切_${file.value.name.split('.')[0]}_${index}.png`;
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
};
</script>

<template>
  <div id="app-container">
    <header class="app-header">
      <h1>书画裁切工具</h1>
      <p>上传图片以自动检测并裁切书画作品。</p>
    </header>

    <main class="main-content">
      <div class="controls">
        <label for="file-upload" class="file-upload-label">
          {{ file ? '更换图片' : '选择图片' }}
        </label>
        <input id="file-upload" type="file" @change="handleFileChange" accept="image/*" />
        
        <button @click="runDetection" :disabled="!file || isLoading" class="refresh-btn">
          刷新检测
        </button>

        <div class="threshold-control">
          <label for="threshold">检测置信度阈值: {{ threshold }}</label>
          <input id="threshold" type="range" v-model.number="threshold" min="0.1" max="1" step="0.05" />
        </div>
        
        <div class="domain-control">
          <label for="service-domain">服务域名:</label>
          <input id="service-domain" list="domain-list" v-model.trim="serviceDomain" placeholder="输入或选择域名" />
          <datalist id="domain-list">
            <option v-for="domain in domainList" :key="domain" :value="domain"></option>
          </datalist>
        </div>
      </div>

      <div v-if="isLoading" class="status-indicator">
        <div class="loader"></div>
        <span>检测中...</span>
      </div>
      <div v-if="error" class="status-indicator error-message">
        <span>{{ error }}</span>
        <button @click="runDetection" class="retry-btn">重试</button>
      </div>

      <div v-if="imageUrl" class="results-area">
        <div class="image-display">
          <div class="image-preview-container">
            <img ref="imagePreview" :src="imageUrl" @load="onImageLoad" alt="图片预览" />
            <div v-for="(result, index) in results" :key="index" class="bbox" :style="getBboxStyle(result.bbox)">
              <span class="bbox-label">{{ result.class_name }} ({{ result.confidence.toFixed(2) }})</span>
            </div>
          </div>
        </div>

        <div v-if="results.length > 0" class="detections-list">
          <h3>检测结果 ({{ results.length }})</h3>
          <ul>
            <li v-for="(result, index) in results" :key="index">
              <div class="detection-info">
                <span class="class-name">{{ result.class_name }}</span>
                <span class="confidence">置信度: {{ result.confidence.toFixed(2) }}</span>
              </div>
              <button @click="downloadCrop(result.bbox, index)" class="download-btn">下载</button>
            </li>
          </ul>
        </div>
      </div>
      <div v-else class="placeholder">
        <p>请选择一张图片开始。</p>
      </div>
    </main>
  </div>
</template>

<style>
:root {
  --primary-color: #42b883;
  --secondary-color: #35495e;
  --background-color: #f4f7f6;
  --text-color: #333;
  --light-gray: #e0e0e0;
  --white: #ffffff;
  --danger-color: #e74c3c;
}

body {
  font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
  margin: 0;
  background-color: var(--background-color);
  color: var(--text-color);
}

#app-container {
  max-width: 1200px;
  margin: 0 auto;
  padding: 1rem 2rem;
}

.app-header {
  text-align: center;
  margin-bottom: 2rem;
  border-bottom: 1px solid var(--light-gray);
  padding-bottom: 1rem;
}

.app-header h1 {
  color: var(--secondary-color);
  margin: 0;
}

.app-header p {
  color: #666;
}

.main-content {
  display: flex;
  flex-direction: column;
  gap: 2rem;
}

.controls {
  display: flex;
  flex-wrap: wrap; /* Allow wrapping on smaller screens */
  justify-content: center;
  align-items: center;
  gap: 1.5rem; /* Adjusted gap */
  padding: 1.5rem;
  background-color: var(--white);
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

input[type="file"] {
  display: none;
}

.file-upload-label {
  background-color: var(--primary-color);
  color: var(--white);
  padding: 12px 20px;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s;
  white-space: nowrap;
}

.file-upload-label:hover {
  background-color: #36a873;
}

.refresh-btn {
  background-color: var(--secondary-color);
  color: var(--white);
  padding: 12px 20px;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  border: none;
  transition: background-color 0.3s;
}

.refresh-btn:hover:not(:disabled) {
  background-color: #2c3e50;
}

.refresh-btn:disabled {
  background-color: var(--light-gray);
  cursor: not-allowed;
}


.threshold-control, .domain-control {
  display: flex;
  align-items: center;
  gap: 0.5rem; /* Tighter gap for label and input */
  color: var(--secondary-color);
}

.domain-control input {
  padding: 8px 12px;
  border: 1px solid var(--light-gray);
  border-radius: 5px;
  min-width: 280px; /* Give it some base width */
  font-size: 1em;
}

input[type="range"] {
  cursor: pointer;
}

.status-indicator {
  text-align: center;
  padding: 1rem;
  border-radius: 8px;
  font-weight: 500;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 1rem; /* Increased gap */
}

.error-message {
  color: var(--danger-color);
  background-color: #fbecec;
  justify-content: space-between; /* Space out text and button */
}

.retry-btn {
  background-color: var(--primary-color);
  color: var(--white);
  border: none;
  padding: 8px 16px;
  border-radius: 5px;
  cursor: pointer;
  font-weight: bold;
  transition: background-color 0.3s;
  white-space: nowrap;
}

.retry-btn:hover {
  background-color: #36a873;
}

.loader {
  border: 4px solid #f3f3f3;
  border-top: 4px solid var(--primary-color);
  border-radius: 50%;
  width: 20px;
  height: 20px;
  animation: spin 1s linear infinite;
}

@keyframes spin {
  0% {
    transform: rotate(0deg);
  }

  100% {
    transform: rotate(360deg);
  }
}

.results-area {
  display: grid;
  grid-template-columns: 2fr 1fr;
  gap: 2rem;
  align-items: flex-start;
}

.image-display {
  background-color: var(--white);
  padding: 1rem;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
}

.image-preview-container {
  position: relative;
  max-width: 100%;
  display: inline-block;
  /* Make container shrink to fit the image */
}

.image-preview-container img {
  max-width: 100%;
  max-height: 70vh;
  /* Set a maximum height */
  height: auto;
  width: auto;
  /* Maintain aspect ratio */
  display: block;
  border-radius: 4px;
  margin: 0 auto;
  /* Center the image */
}

.bbox {
  position: absolute;
  border: 2px solid var(--primary-color);
  box-sizing: border-box;
  transition: all 0.2s ease-in-out;
}

.bbox:hover {
  border-width: 3px;
  box-shadow: 0 0 10px rgba(66, 184, 131, 0.5);
}

.bbox-label {
  position: absolute;
  bottom: 100%;
  left: -2px;
  margin-bottom: 4px;
  background-color: var(--primary-color);
  color: white;
  padding: 3px 6px;
  font-size: 12px;
  font-weight: bold;
  white-space: nowrap;
  border-radius: 3px;
}

.detections-list {
  background-color: var(--white);
  padding: 1.5rem;
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  max-height: 60vh;
  overflow-y: auto;
}

.detections-list h3 {
  margin-top: 0;
  color: var(--secondary-color);
  border-bottom: 1px solid var(--light-gray);
  padding-bottom: 0.5rem;
}

.detections-list ul {
  list-style: none;
  padding: 0;
  margin: 0;
}

.detections-list li {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 0.75rem 0;
  border-bottom: 1px solid var(--light-gray);
}

.detections-list li:last-child {
  border-bottom: none;
}

.detection-info {
  display: flex;
  flex-direction: column;
}

.class-name {
  font-weight: bold;
}

.confidence {
  font-size: 0.9em;
  color: #555;
}

.download-btn {
  background-color: var(--secondary-color);
  color: var(--white);
  border: none;
  padding: 8px 12px;
  border-radius: 5px;
  cursor: pointer;
  transition: background-color 0.3s;
}

.download-btn:hover {
  background-color: #2c3e50;
}

.placeholder {
  text-align: center;
  padding: 4rem;
  background-color: var(--white);
  border-radius: 8px;
  box-shadow: 0 2px 8px rgba(0, 0, 0, 0.05);
  color: #888;
}

@media (max-width: 992px) { /* Adjusted breakpoint for better responsiveness */
  .results-area {
    grid-template-columns: 1fr;
  }

  .controls {
    flex-direction: column;
    gap: 1.5rem;
  }

  .domain-control input {
    width: 100%;
    min-width: unset;
  }
}
</style>