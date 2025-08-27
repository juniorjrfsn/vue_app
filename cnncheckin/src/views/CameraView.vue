<template>
  <div class="camera-container">
    <div class="camera-header">
      <h1>📸 Câmera</h1>
      <p>Tire fotos diretamente do navegador</p>
    </div>

    <!-- Área da câmera -->
    <div class="camera-section">
      <div class="video-container" v-if="!showCapturedPhoto">
        <video 
          ref="videoElement" 
          :class="{ 'mirrored': isFrontCamera }"
          autoplay 
          muted 
          playsinline
        ></video>
        
        <!-- Overlay com controles -->
        <div class="camera-overlay">
          <div class="top-controls">
            <button 
              @click="toggleCamera" 
              class="control-btn"
              :disabled="!hasMultipleCameras"
              title="Alternar câmera"
            >
              🔄
            </button>
            <button 
              @click="toggleFlash" 
              class="control-btn"
              :class="{ active: flashEnabled }"
              title="Flash"
            >
              {{ flashEnabled ? '⚡' : '🔦' }}
            </button>
          </div>
          
          <div class="bottom-controls">
            <div class="photos-count" v-if="capturedPhotos.length > 0">
              <span>{{ capturedPhotos.length }} foto(s)</span>
            </div>
            
            <button 
              @click="capturePhoto" 
              class="capture-btn"
              :disabled="!cameraActive"
            >
              <div class="capture-inner"></div>
            </button>
            
            <button 
              @click="showGallery = true" 
              class="gallery-btn"
              v-if="capturedPhotos.length > 0"
            >
              🖼️
            </button>
          </div>
        </div>
      </div>

      <!-- Foto capturada -->
      <div class="captured-photo" v-if="showCapturedPhoto">
        <img :src="lastCapturedPhoto" alt="Foto capturada" />
        <div class="photo-actions">
          <button @click="retakePhoto" class="action-btn secondary">
            🔄 Tirar Novamente
          </button>
          <button @click="savePhoto" class="action-btn primary">
            💾 Salvar Foto
          </button>
        </div>
      </div>
    </div>

    <!-- Status da câmera -->
    <div class="camera-status" v-if="statusMessage">
      <p :class="statusType">{{ statusMessage }}</p>
    </div>

    <!-- Botões de controle -->
    <div class="controls" v-if="!showCapturedPhoto">
      <button 
        @click="startCamera" 
        v-if="!cameraActive"
        class="control-button primary"
      >
        📹 Iniciar Câmera
      </button>
      
      <button 
        @click="stopCamera" 
        v-if="cameraActive"
        class="control-button secondary"
      >
        ⏹️ Parar Câmera
      </button>
    </div>

    <!-- Modal da galeria -->
    <div class="gallery-modal" v-if="showGallery" @click.self="showGallery = false">
      <div class="gallery-content">
        <div class="gallery-header">
          <h2>Galeria de Fotos</h2>
          <button @click="showGallery = false" class="close-btn">✕</button>
        </div>
        
        <div class="gallery-grid" v-if="capturedPhotos.length > 0">
          <div 
            v-for="(photo, index) in capturedPhotos" 
            :key="index"
            class="gallery-item"
          >
            <img :src="photo.url" :alt="`Foto ${index + 1}`" />
            <div class="photo-info">
              <small>{{ photo.timestamp }}</small>
              <div class="photo-actions-small">
                <button @click="downloadPhoto(photo, index)" class="small-btn">⬇️</button>
                <button @click="deletePhoto(index)" class="small-btn delete">🗑️</button>
              </div>
            </div>
          </div>
        </div>
        
        <div v-else class="no-photos">
          <p>Nenhuma foto capturada ainda</p>
        </div>
        
        <div class="gallery-actions" v-if="capturedPhotos.length > 0">
          <button @click="downloadAllPhotos" class="gallery-btn-action">
            📁 Baixar Todas
          </button>
          <button @click="clearAllPhotos" class="gallery-btn-action danger">
            🗑️ Limpar Todas
          </button>
        </div>
      </div>
    </div>
  </div>
</template>

<script setup lang="ts">
import { ref, onMounted, onBeforeUnmount } from 'vue'

// Tipos
interface CapturedPhoto {
  url: string
  timestamp: string
  blob: Blob
}

// Referencias reativas
const videoElement = ref<HTMLVideoElement | null>(null)
const cameraActive = ref(false)
const showCapturedPhoto = ref(false)
const showGallery = ref(false)
const lastCapturedPhoto = ref('')
const capturedPhotos = ref<CapturedPhoto[]>([])
const statusMessage = ref('')
const statusType = ref<'success' | 'error' | 'info'>('info')
const isFrontCamera = ref(true)
const hasMultipleCameras = ref(false)
const flashEnabled = ref(false)

// Stream da câmera
let currentStream: MediaStream | null = null
let currentDeviceId: string | null = null
let availableCameras: MediaDeviceInfo[] = []

// Funções da câmera
const getCameraDevices = async (): Promise<MediaDeviceInfo[]> => {
  try {
    const devices = await navigator.mediaDevices.enumerateDevices()
    return devices.filter(device => device.kind === 'videoinput')
  } catch (error) {
    console.error('Erro ao obter dispositivos:', error)
    return []
  }
}

const startCamera = async (deviceId?: string) => {
  try {
    setStatus('Iniciando câmera...', 'info')
    
    // Parar stream anterior se existir
    if (currentStream) {
      currentStream.getTracks().forEach(track => track.stop())
    }

    // Configurações da câmera
    const constraints: MediaStreamConstraints = {
      video: {
        width: { ideal: 1280, max: 1920 },
        height: { ideal: 720, max: 1080 },
        facingMode: deviceId ? undefined : (isFrontCamera.value ? 'user' : 'environment'),
        deviceId: deviceId ? { exact: deviceId } : undefined
      }
    }

    const stream = await navigator.mediaDevices.getUserMedia(constraints)
    
    if (videoElement.value) {
      videoElement.value.srcObject = stream
      currentStream = stream
      cameraActive.value = true
      setStatus('Câmera ativa', 'success')
      
      // Verificar múltiplas câmeras
      availableCameras = await getCameraDevices()
      hasMultipleCameras.value = availableCameras.length > 1
    }
  } catch (error: any) {
    console.error('Erro ao acessar câmera:', error)
    let message = 'Erro ao acessar a câmera'
    
    if (error.name === 'NotAllowedError') {
      message = 'Permissão para câmera negada. Permita o acesso para continuar.'
    } else if (error.name === 'NotFoundError') {
      message = 'Nenhuma câmera encontrada no dispositivo.'
    } else if (error.name === 'NotReadableError') {
      message = 'Câmera está sendo usada por outro aplicativo.'
    }
    
    setStatus(message, 'error')
  }
}

const stopCamera = () => {
  if (currentStream) {
    currentStream.getTracks().forEach(track => track.stop())
    currentStream = null
  }
  
  if (videoElement.value) {
    videoElement.value.srcObject = null
  }
  
  cameraActive.value = false
  setStatus('Câmera desligada', 'info')
}

const toggleCamera = async () => {
  if (!hasMultipleCameras.value || availableCameras.length < 2) return
  
  const currentIndex = availableCameras.findIndex(camera => camera.deviceId === currentDeviceId)
  const nextIndex = (currentIndex + 1) % availableCameras.length
  const nextCamera = availableCameras[nextIndex]
  
  currentDeviceId = nextCamera.deviceId
  isFrontCamera.value = nextCamera.label.toLowerCase().includes('front') || 
                        nextCamera.label.toLowerCase().includes('user')
  
  await startCamera(nextCamera.deviceId)
}

const toggleFlash = () => {
  flashEnabled.value = !flashEnabled.value
  
  if (currentStream) {
    const videoTrack = currentStream.getVideoTracks()[0]
    if (videoTrack && 'torch' in videoTrack.getCapabilities()) {
      videoTrack.applyConstraints({
        advanced: [{ torch: flashEnabled.value } as any]
      }).catch(() => {
        setStatus('Flash não suportado neste dispositivo', 'error')
        flashEnabled.value = false
      })
    } else {
      setStatus('Flash não suportado neste dispositivo', 'error')
      flashEnabled.value = false
    }
  }
}

const capturePhoto = () => {
  if (!videoElement.value || !cameraActive.value) return
  
  const canvas = document.createElement('canvas')
  const context = canvas.getContext('2d')
  
  if (!context) return
  
  // Definir dimensões do canvas
  canvas.width = videoElement.value.videoWidth
  canvas.height = videoElement.value.videoHeight
  
  // Espelhar imagem se for câmera frontal
  if (isFrontCamera.value) {
    context.scale(-1, 1)
    context.translate(-canvas.width, 0)
  }
  
  // Desenhar frame atual do vídeo
  context.drawImage(videoElement.value, 0, 0)
  
  // Converter para blob
  canvas.toBlob((blob) => {
    if (blob) {
      const url = URL.createObjectURL(blob)
      const timestamp = new Date().toLocaleString('pt-BR')
      
      const photo: CapturedPhoto = {
        url,
        timestamp,
        blob
      }
      
      capturedPhotos.value.push(photo)
      lastCapturedPhoto.value = url
      showCapturedPhoto.value = true
      
      setStatus('Foto capturada!', 'success')
    }
  }, 'image/jpeg', 0.9)
}

const retakePhoto = () => {
  showCapturedPhoto.value = false
  // Remover última foto se não foi salva
  if (capturedPhotos.value.length > 0) {
    const lastPhoto = capturedPhotos.value.pop()
    if (lastPhoto) {
      URL.revokeObjectURL(lastPhoto.url)
    }
  }
}

const savePhoto = () => {
  if (lastCapturedPhoto.value) {
    const link = document.createElement('a')
    link.href = lastCapturedPhoto.value
    link.download = `foto_${new Date().toISOString().slice(0, 19).replace(/:/g, '-')}.jpg`
    link.click()
    
    setStatus('Foto salva!', 'success')
    showCapturedPhoto.value = false
  }
}

const downloadPhoto = (photo: CapturedPhoto, index: number) => {
  const link = document.createElement('a')
  link.href = photo.url
  link.download = `foto_${index + 1}_${photo.timestamp.replace(/[/: ]/g, '_')}.jpg`
  link.click()
}

const deletePhoto = (index: number) => {
  if (confirm('Tem certeza que deseja excluir esta foto?')) {
    const photo = capturedPhotos.value[index]
    URL.revokeObjectURL(photo.url)
    capturedPhotos.value.splice(index, 1)
  }
}

const downloadAllPhotos = () => {
  capturedPhotos.value.forEach((photo, index) => {
    setTimeout(() => downloadPhoto(photo, index), index * 100)
  })
}

const clearAllPhotos = () => {
  if (confirm('Tem certeza que deseja excluir todas as fotos?')) {
    capturedPhotos.value.forEach(photo => {
      URL.revokeObjectURL(photo.url)
    })
    capturedPhotos.value = []
  }
}

const setStatus = (message: string, type: 'success' | 'error' | 'info') => {
  statusMessage.value = message
  statusType.value = type
  
  setTimeout(() => {
    statusMessage.value = ''
  }, 3000)
}

// Lifecycle hooks
onMounted(async () => {
  // Verificar suporte da câmera
  if (!navigator.mediaDevices || !navigator.mediaDevices.getUserMedia) {
    setStatus('Câmera não suportada neste navegador', 'error')
    return
  }
  
  // Iniciar câmera automaticamente
  await startCamera()
})

onBeforeUnmount(() => {
  stopCamera()
  
  // Limpar URLs de objetos
  capturedPhotos.value.forEach(photo => {
    URL.revokeObjectURL(photo.url)
  })
})
</script>

<style scoped>
.camera-container {
  max-width: 800px;
  margin: 0 auto;
  padding: 20px;
  font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif;
}

.camera-header {
  text-align: center;
  margin-bottom: 30px;
}

.camera-header h1 {
  font-size: 2.5rem;
  margin: 0 0 10px 0;
  color: #333;
}

.camera-header p {
  color: #666;
  font-size: 1.1rem;
  margin: 0;
}

.camera-section {
  position: relative;
  margin-bottom: 20px;
}

.video-container {
  position: relative;
  border-radius: 20px;
  overflow: hidden;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  background: #000;
}

video {
  width: 100%;
  height: auto;
  display: block;
  max-height: 60vh;
  object-fit: cover;
}

.mirrored {
  transform: scaleX(-1);
}

.camera-overlay {
  position: absolute;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  pointer-events: none;
}

.top-controls {
  position: absolute;
  top: 20px;
  right: 20px;
  display: flex;
  gap: 10px;
  pointer-events: all;
}

.bottom-controls {
  position: absolute;
  bottom: 30px;
  left: 0;
  right: 0;
  display: flex;
  justify-content: center;
  align-items: center;
  gap: 30px;
  pointer-events: all;
}

.control-btn {
  width: 50px;
  height: 50px;
  border: none;
  border-radius: 50%;
  background: rgba(0,0,0,0.7);
  color: white;
  font-size: 1.5rem;
  cursor: pointer;
  transition: all 0.3s;
  backdrop-filter: blur(10px);
}

.control-btn:hover:not(:disabled) {
  background: rgba(0,0,0,0.9);
  transform: scale(1.1);
}

.control-btn:disabled {
  opacity: 0.5;
  cursor: not-allowed;
}

.control-btn.active {
  background: #ffeb3b;
  color: #333;
}

.capture-btn {
  width: 80px;
  height: 80px;
  border: 4px solid white;
  border-radius: 50%;
  background: transparent;
  cursor: pointer;
  transition: transform 0.2s;
  position: relative;
}

.capture-btn:hover:not(:disabled) {
  transform: scale(1.1);
}

.capture-btn:active {
  transform: scale(0.95);
}

.capture-inner {
  width: 60px;
  height: 60px;
  background: white;
  border-radius: 50%;
  margin: auto;
  transition: background 0.2s;
}

.capture-btn:disabled .capture-inner {
  background: #ccc;
}

.photos-count {
  background: rgba(0,0,0,0.7);
  color: white;
  padding: 8px 16px;
  border-radius: 20px;
  font-size: 0.9rem;
  backdrop-filter: blur(10px);
}

.gallery-btn {
  width: 50px;
  height: 50px;
  border: 2px solid white;
  border-radius: 50%;
  background: rgba(0,0,0,0.7);
  color: white;
  font-size: 1.5rem;
  cursor: pointer;
  transition: all 0.3s;
  backdrop-filter: blur(10px);
}

.gallery-btn:hover {
  background: rgba(255,255,255,0.2);
}

.captured-photo {
  text-align: center;
}

.captured-photo img {
  max-width: 100%;
  height: auto;
  border-radius: 20px;
  box-shadow: 0 10px 30px rgba(0,0,0,0.3);
  margin-bottom: 20px;
}

.photo-actions {
  display: flex;
  gap: 15px;
  justify-content: center;
}

.action-btn {
  padding: 12px 24px;
  border: none;
  border-radius: 25px;
  font-size: 1rem;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: 500;
}

.action-btn.primary {
  background: #4CAF50;
  color: white;
}

.action-btn.secondary {
  background: #f0f0f0;
  color: #333;
}

.action-btn:hover {
  transform: translateY(-2px);
  box-shadow: 0 5px 15px rgba(0,0,0,0.2);
}

.camera-status {
  text-align: center;
  margin: 20px 0;
}

.camera-status p {
  padding: 12px 20px;
  border-radius: 8px;
  margin: 0;
  font-weight: 500;
}

.camera-status .success {
  background: #d4edda;
  color: #155724;
  border: 1px solid #c3e6cb;
}

.camera-status .error {
  background: #f8d7da;
  color: #721c24;
  border: 1px solid #f5c6cb;
}

.camera-status .info {
  background: #d1ecf1;
  color: #0c5460;
  border: 1px solid #bee5eb;
}

.controls {
  display: flex;
  justify-content: center;
  gap: 15px;
}

.control-button {
  padding: 15px 30px;
  border: none;
  border-radius: 25px;
  font-size: 1.1rem;
  cursor: pointer;
  transition: all 0.3s;
  font-weight: 500;
}

.control-button.primary {
  background: #2196F3;
  color: white;
}

.control-button.secondary {
  background: #f0f0f0;
  color: #333;
}

.control-button:hover {
  transform: translateY(-2px);
  box-shadow: 0 8px 20px rgba(0,0,0,0.2);
}

/* Modal da galeria */
.gallery-modal {
  position: fixed;
  top: 0;
  left: 0;
  right: 0;
  bottom: 0;
  background: rgba(0,0,0,0.8);
  display: flex;
  justify-content: center;
  align-items: center;
  z-index: 1000;
  padding: 20px;
}

.gallery-content {
  background: white;
  border-radius: 20px;
  max-width: 90vw;
  max-height: 90vh;
  overflow: auto;
  padding: 0;
}

.gallery-header {
  display: flex;
  justify-content: space-between;
  align-items: center;
  padding: 20px 30px;
  border-bottom: 1px solid #eee;
  position: sticky;
  top: 0;
  background: white;
  border-radius: 20px 20px 0 0;
}

.gallery-header h2 {
  margin: 0;
  color: #333;
}

.close-btn {
  background: none;
  border: none;
  font-size: 1.5rem;
  cursor: pointer;
  color: #666;
  width: 40px;
  height: 40px;
  border-radius: 50%;
  display: flex;
  align-items: center;
  justify-content: center;
}

.close-btn:hover {
  background: #f0f0f0;
}

.gallery-grid {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(200px, 1fr));
  gap: 20px;
  padding: 20px;
}

.gallery-item {
  position: relative;
  border-radius: 10px;
  overflow: hidden;
  box-shadow: 0 4px 10px rgba(0,0,0,0.1);
}

.gallery-item img {
  width: 100%;
  height: 150px;
  object-fit: cover;
}

.photo-info {
  padding: 10px;
  background: white;
  display: flex;
  justify-content: space-between;
  align-items: center;
}

.photo-actions-small {
  display: flex;
  gap: 5px;
}

.small-btn {
  background: none;
  border: none;
  font-size: 1rem;
  cursor: pointer;
  padding: 5px;
  border-radius: 4px;
  transition: background 0.2s;
}

.small-btn:hover {
  background: #f0f0f0;
}

.small-btn.delete:hover {
  background: #ffebee;
  color: #d32f2f;
}

.no-photos {
  text-align: center;
  padding: 50px 20px;
  color: #666;
}

.gallery-actions {
  padding: 20px;
  border-top: 1px solid #eee;
  display: flex;
  justify-content: center;
  gap: 15px;
  background: #fafafa;
}

.gallery-btn-action {
  padding: 10px 20px;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  font-weight: 500;
  transition: all 0.3s;
}

.gallery-btn-action:not(.danger) {
  background: #2196F3;
  color: white;
}

.gallery-btn-action.danger {
  background: #f44336;
  color: white;
}

.gallery-btn-action:hover {
  transform: translateY(-1px);
  box-shadow: 0 4px 10px rgba(0,0,0,0.2);
}

/* Responsividade */
@media (max-width: 768px) {
  .camera-container {
    padding: 15px;
  }
  
  .camera-header h1 {
    font-size: 2rem;
  }
  
  .capture-btn {
    width: 70px;
    height: 70px;
  }
  
  .capture-inner {
    width: 50px;
    height: 50px;
  }
  
  .gallery-grid {
    grid-template-columns: repeat(auto-fill, minmax(150px, 1fr));
    gap: 15px;
    padding: 15px;
  }
  
  .gallery-item img {
    height: 120px;
  }
}

@media (max-width: 480px) {
  .bottom-controls {
    gap: 20px;
  }
  
  .top-controls {
    top: 15px;
    right: 15px;
  }
  
  .control-btn {
    width: 40px;
    height: 40px;
    font-size: 1.2rem;
  }
  
  .gallery-btn {
    width: 40px;
    height: 40px;
    font-size: 1.2rem;
  }
}
</style>