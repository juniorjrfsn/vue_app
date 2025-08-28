<template>
  <div class="camera-container">
    <div class="camera-header">
      <h1>📸 Câmera</h1>
      <p>Tire fotos diretamente do navegador</p>
    </div>

    <!-- Debug Info -->
    <div class="debug-info" v-if="debugMode">
      <h3>Debug Info:</h3>
      <p>Navegador suportado: {{ browserSupported ? 'Sim' : 'Não' }}</p>
      <p>HTTPS/Localhost: {{ isSecureContext ? 'Sim' : 'Não' }}</p>
      <p>Dispositivos encontrados: {{ availableCameras.length }}</p>
      <p>Câmera ativa: {{ cameraActive ? 'Sim' : 'Não' }}</p>
      <p>Último erro: {{ lastError }}</p>
    </div>

    <!-- Área da câmera -->
    <div class="camera-section">
      <div class="video-container" v-if="!showCapturedPhoto && cameraActive">
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
              @click="toggleDebug" 
              class="control-btn"
              title="Debug"
            >
              🐛
            </button>
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

      <!-- Placeholder quando câmera não está ativa -->
      <div class="camera-placeholder" v-if="!cameraActive && !showCapturedPhoto">
        <div class="placeholder-content">
          <div class="camera-icon">📷</div>
          <h3>Câmera não iniciada</h3>
          <p v-if="!browserSupported">Seu navegador não suporta acesso à câmera</p>
          <p v-else-if="!isSecureContext">A câmera só funciona em HTTPS ou localhost</p>
          <p v-else>Clique no botão abaixo para iniciar</p>
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
    <div class="controls">
      <button 
        @click="requestPermissions" 
        v-if="!permissionGranted && !cameraActive"
        class="control-button primary"
      >
        🔐 Solicitar Permissão
      </button>
      
      <button 
        @click="startCamera" 
        v-if="!cameraActive && permissionGranted"
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

      <button 
        @click="testCamera" 
        class="control-button info"
      >
        🔧 Testar Sistema
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
            📦 Baixar Todas
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
const permissionGranted = ref(false)
const browserSupported = ref(false)
const isSecureContext = ref(false)
const debugMode = ref(false)
const lastError = ref('')

// Stream da câmera
let currentStream: MediaStream | null = null
let currentDeviceId: string | null = null
let availableCameras: MediaDeviceInfo[] = []

// Funções de diagnóstico
const checkBrowserSupport = (): boolean => {
  const supported = !!(
    navigator.mediaDevices && 
    navigator.mediaDevices.getUserMedia &&
    window.MediaStream
  )
  browserSupported.value = supported
  return supported
}

const checkSecureContext = (): boolean => {
  const secure = window.isSecureContext || 
                 location.protocol === 'https:' || 
                 location.hostname === 'localhost' ||
                 location.hostname === '127.0.0.1'
  isSecureContext.value = secure
  return secure
}

const testCamera = async () => {
  setStatus('Testando sistema...', 'info')
  
  // Verificar suporte do navegador
  if (!checkBrowserSupport()) {
    setStatus('❌ Navegador não suporta acesso à câmera', 'error')
    return
  }

  // Verificar contexto seguro
  if (!checkSecureContext()) {
    setStatus('❌ Câmera só funciona em HTTPS ou localhost', 'error')
    return
  }

  // Verificar permissões
  try {
    const permissionStatus = await navigator.permissions.query({ name: 'camera' as PermissionName })
    console.log('Permission status:', permissionStatus.state)
    
    if (permissionStatus.state === 'denied') {
      setStatus('❌ Permissão para câmera foi negada', 'error')
      return
    }
  } catch (error) {
    console.log('Não foi possível verificar permissões:', error)
  }

  // Listar dispositivos
  try {
    const devices = await getCameraDevices()
    console.log('Dispositivos encontrados:', devices)
    
    if (devices.length === 0) {
      setStatus('❌ Nenhuma câmera encontrada', 'error')
      return
    }
    
    setStatus(`✅ Sistema OK! ${devices.length} câmera(s) encontrada(s)`, 'success')
  } catch (error: any) {
    console.error('Erro ao testar:', error)
    lastError.value = error.message || error.toString()
    setStatus(`❌ Erro no teste: ${error.message}`, 'error')
  }
}

const requestPermissions = async () => {
  setStatus('Solicitando permissão...', 'info')
  
  try {
    // Tentar acessar a câmera brevemente para solicitar permissão
    const stream = await navigator.mediaDevices.getUserMedia({ 
      video: { width: 320, height: 240 } 
    })
    
    // Parar o stream imediatamente
    stream.getTracks().forEach(track => track.stop())
    
    permissionGranted.value = true
    setStatus('✅ Permissão concedida!', 'success')
    
  } catch (error: any) {
    console.error('Erro ao solicitar permissão:', error)
    lastError.value = error.message || error.toString()
    
    let message = 'Erro ao solicitar permissão para câmera'
    
    if (error.name === 'NotAllowedError') {
      message = '❌ Permissão para câmera foi negada. Verifique as configurações do navegador.'
    } else if (error.name === 'NotFoundError') {
      message = '❌ Nenhuma câmera encontrada no dispositivo.'
    } else if (error.name === 'NotReadableError') {
      message = '❌ Câmera está sendo usada por outro aplicativo.'
    } else if (error.name === 'OverconstrainedError') {
      message = '❌ Configurações da câmera não são suportadas.'
    } else if (error.name === 'SecurityError') {
      message = '❌ Erro de segurança. Verifique se está usando HTTPS.'
    }
    
    setStatus(message, 'error')
  }
}

const toggleDebug = () => {
  debugMode.value = !debugMode.value
}

// Funções da câmera
const getCameraDevices = async (): Promise<MediaDeviceInfo[]> => {
  try {
    const devices = await navigator.mediaDevices.enumerateDevices()
    const cameras = devices.filter(device => device.kind === 'videoinput')
    availableCameras = cameras
    return cameras
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

    // Configurações da câmera com fallbacks
    const constraints: MediaStreamConstraints = {
      video: {
        width: { ideal: 1280, min: 640, max: 1920 },
        height: { ideal: 720, min: 480, max: 1080 },
        frameRate: { ideal: 30, min: 15, max: 60 },
        facingMode: deviceId ? undefined : (isFrontCamera.value ? 'user' : 'environment'),
        deviceId: deviceId ? { exact: deviceId } : undefined
      }
    }

    console.log('Tentando iniciar câmera com constraints:', constraints)
    const stream = await navigator.mediaDevices.getUserMedia(constraints)
    
    if (videoElement.value) {
      videoElement.value.srcObject = stream
      currentStream = stream
      cameraActive.value = true
      permissionGranted.value = true
      
      // Aguardar o vídeo carregar
      await new Promise<void>((resolve) => {
        if (videoElement.value) {
          videoElement.value.onloadedmetadata = () => resolve()
        }
      })
      
      setStatus('✅ Câmera ativa', 'success')
      
      // Verificar múltiplas câmeras
      const cameras = await getCameraDevices()
      hasMultipleCameras.value = cameras.length > 1
      
      console.log('Câmera iniciada com sucesso')
    }
  } catch (error: any) {
    console.error('Erro ao acessar câmera:', error)
    lastError.value = error.message || error.toString()
    
    let message = 'Erro ao acessar a câmera'
    
    if (error.name === 'NotAllowedError') {
      message = '❌ Permissão para câmera negada. Clique em "Solicitar Permissão" primeiro.'
      permissionGranted.value = false
    } else if (error.name === 'NotFoundError') {
      message = '❌ Nenhuma câmera encontrada no dispositivo.'
    } else if (error.name === 'NotReadableError') {
      message = '❌ Câmera está sendo usada por outro aplicativo.'
    } else if (error.name === 'OverconstrainedError') {
      message = '❌ Configurações da câmera não são suportadas. Tentando configuração básica...'
      // Tentar novamente com configurações mais simples
      setTimeout(() => startCameraBasic(), 1000)
    } else if (error.name === 'SecurityError') {
      message = '❌ Erro de segurança. Verifique se está usando HTTPS.'
    }
    
    setStatus(message, 'error')
  }
}

const startCameraBasic = async () => {
  try {
    setStatus('Tentando configuração básica...', 'info')
    
    const stream = await navigator.mediaDevices.getUserMedia({
      video: true
    })
    
    if (videoElement.value) {
      videoElement.value.srcObject = stream
      currentStream = stream
      cameraActive.value = true
      setStatus('✅ Câmera ativa (modo básico)', 'success')
    }
  } catch (error: any) {
    console.error('Erro mesmo com configuração básica:', error)
    setStatus('❌ Não foi possível iniciar a câmera', 'error')
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
  flashEnabled.value = false
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
    if (videoTrack && videoTrack.getCapabilities && 'torch' in videoTrack.getCapabilities()) {
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
  
  try {
    const canvas = document.createElement('canvas')
    const context = canvas.getContext('2d')
    
    if (!context) {
      setStatus('Erro ao criar canvas', 'error')
      return
    }
    
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
        
        setStatus('✅ Foto capturada!', 'success')
      } else {
        setStatus('Erro ao capturar foto', 'error')
      }
    }, 'image/jpeg', 0.9)
  } catch (error: any) {
    console.error('Erro ao capturar foto:', error)
    setStatus('Erro ao capturar foto', 'error')
  }
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
    
    setStatus('✅ Foto salva!', 'success')
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
  }, 5000)
}

// Lifecycle hooks
onMounted(async () => {
  console.log('CameraView montado')
  
  // Verificações iniciais
  checkBrowserSupport()
  checkSecureContext()
  
  if (!browserSupported.value) {
    setStatus('❌ Navegador não suporta acesso à câmera', 'error')
    return
  }
  
  if (!isSecureContext.value) {
    setStatus('❌ Câmera só funciona em HTTPS ou localhost', 'error')
    return
  }
  
  // Verificar permissões existentes
  try {
    const permissionStatus = await navigator.permissions.query({ name: 'camera' as PermissionName })
    if (permissionStatus.state === 'granted') {
      permissionGranted.value = true
      // Auto-iniciar se já tiver permissão
      setTimeout(() => startCamera(), 500)
    }
  } catch (error) {
    console.log('Não foi possível verificar permissões automaticamente')
  }
})

onBeforeUnmount(() => {
  stopCamera()
  
  // Limpar URLs de objetos
  capturedPhotos.value.forEach(photo => {
    URL.revokeObjectURL(photo.url)
  })
})
</script>

 

<style src="@/assets/camera.css" scoped></style>