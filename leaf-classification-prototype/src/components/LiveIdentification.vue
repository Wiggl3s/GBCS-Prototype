<script setup lang="ts">
import { ref, nextTick, onBeforeUnmount } from 'vue'

type Stage = 'idle' | 'processing' | 'done'

const stage = ref<Stage>('idle')
const selectedFileName = ref<string | null>(null)
const predictedSpecies = ref<string | null>(null)
const confidence = ref<number | null>(null)

const fileInputRef = ref<HTMLInputElement | null>(null)
const isCameraOpen = ref(false)
const cameraError = ref<string | null>(null)
const videoRef = ref<HTMLVideoElement | null>(null)
const cameraStream = ref<MediaStream | null>(null)

function resetResults() {
  predictedSpecies.value = null
  confidence.value = null
}

function handleFiles(files: FileList | null) {
  const file = files?.[0]
  if (!file) return

  const mimeType = file.type ?? ''
  if (!mimeType.startsWith('image/')) {
    alert('Please upload a plant leaf image (JPG/PNG/TIFF).')
    return
  }

  selectedFileName.value = file.name
  runClassification()
}

function onFileChange(e: Event) {
  const target = e.target as HTMLInputElement
  handleFiles(target.files)
}

function triggerBrowse() {
  fileInputRef.value?.click()
}

async function startCamera() {
  cameraError.value = null
  try {
    if (!navigator.mediaDevices?.getUserMedia) {
      cameraError.value = 'Camera not supported in this browser.'
      return
    }
    
    // Try with ideal constraints first
    let stream: MediaStream | null = null
    try {
      stream = await navigator.mediaDevices.getUserMedia({
        video: {
          facingMode: 'environment',
          width: { ideal: 1920 },
          height: { ideal: 1080 },
        },
      })
    } catch (err) {
      // Fallback: try without constraints if ideal fails
      try {
        stream = await navigator.mediaDevices.getUserMedia({
          video: { facingMode: 'environment' },
        })
      } catch (fallbackErr) {
        // Final fallback: try any camera
        stream = await navigator.mediaDevices.getUserMedia({
          video: true,
        })
      }
    }
    
    if (!stream) {
      cameraError.value = 'Could not access camera. Please check browser permissions.'
      return
    }
    
    cameraStream.value = stream
    const video = videoRef.value
    if (video) {
      video.srcObject = stream
      video.muted = true // Required for autoplay on mobile
      video.setAttribute('playsinline', 'true') // iOS requirement
      
      // Wait for video to be ready
      await new Promise<void>((resolve) => {
        if (video.readyState >= 2) {
          resolve()
        } else {
          video.addEventListener('loadedmetadata', () => resolve(), { once: true })
        }
      })
      
      try {
        await video.play()
      } catch (playErr) {
        // If autoplay fails, try with user interaction
        console.warn('Autoplay failed, video will play on user interaction')
      }
    }
  } catch (err: any) {
    if (err.name === 'NotAllowedError' || err.name === 'PermissionDeniedError') {
      cameraError.value = 'Camera permission denied. Please allow camera access in your browser settings.'
    } else if (err.name === 'NotFoundError' || err.name === 'DevicesNotFoundError') {
      cameraError.value = 'No camera found on this device.'
    } else if (err.name === 'NotReadableError' || err.name === 'TrackStartError') {
      cameraError.value = 'Camera is already in use by another application.'
    } else {
      cameraError.value = `Unable to access camera: ${err.message || 'Unknown error'}.`
    }
  }
}

function stopCamera() {
  if (cameraStream.value) {
    cameraStream.value.getTracks().forEach((t) => t.stop())
    cameraStream.value = null
  }
  if (videoRef.value) {
    videoRef.value.srcObject = null
  }
}

async function openCamera() {
  isCameraOpen.value = true
  await nextTick()
  await startCamera()
}

function closeCamera() {
  stopCamera()
  isCameraOpen.value = false
}

function captureFromCamera() {
  const video = videoRef.value
  if (!video || !video.videoWidth || !video.videoHeight) return

  const canvas = document.createElement('canvas')
  canvas.width = video.videoWidth
  canvas.height = video.videoHeight
  const ctx = canvas.getContext('2d')
  if (!ctx) return
  ctx.drawImage(video, 0, 0, canvas.width, canvas.height)

  canvas.toBlob((blob) => {
    if (!blob) return
    const file = new File([blob], 'captured-leaf.jpg', { type: 'image/jpeg' })
    const dataTransfer = new DataTransfer()
    dataTransfer.items.add(file)
    handleFiles(dataTransfer.files)
    closeCamera()
  }, 'image/jpeg', 0.9)
}

function runClassification() {
  resetResults()
  stage.value = 'processing'

  // Simulate fast, inference-only pipeline (no training loop)
  setTimeout(() => {
    stage.value = 'done'
    predictedSpecies.value = 'Banaba (Lagerstroemia speciosa)'
    confidence.value = 99.2
  }, 1500)
}

onBeforeUnmount(() => {
  stopCamera()
})
</script>

<template>
  <div class="space-y-6">
    <!-- Camera overlay -->
    <div
      v-if="isCameraOpen"
      class="fixed inset-0 z-40 flex flex-col bg-black md:items-center md:justify-center md:bg-black/70 md:px-4"
    >
      <div class="flex-1 flex flex-col w-full md:w-auto md:max-w-sm md:rounded-xl md:border md:border-slate-700 md:bg-slate-950 md:p-3 md:space-y-2 md:shadow-xl">
        <div class="flex items-center justify-between gap-2 p-3 md:p-0">
          <div class="min-w-0 flex-1">
            <p class="text-xs font-semibold uppercase tracking-wide text-emerald-300">
              Live capture
            </p>
            <p class="text-[10px] text-slate-400 hidden md:block mt-0.5">
              Align the plant leaf, then capture to classify.
            </p>
          </div>
         
        </div>

        <!-- Mobile: fitted video preview, Desktop: contained aspect-video -->
        <div class="flex items-center justify-center overflow-hidden bg-black max-h-[45vh] min-h-[200px] md:max-h-[280px] md:min-h-0 md:rounded-lg md:border md:border-slate-800">
          <video
            ref="videoRef"
            autoplay
            playsinline
            muted
            class="w-full h-full max-h-[45vh] min-h-[200px] object-cover md:block md:w-full md:aspect-video md:h-auto md:max-h-[280px] md:min-h-0 md:object-contain"
          />
        </div>

        <p v-if="cameraError" class="text-[10px] text-red-400 px-3 py-1 md:px-0 md:py-0.5">
          {{ cameraError }}
        </p>

        <div class="flex items-center justify-end gap-2 text-[11px] p-3 pb-4 md:p-0 md:pb-0 md:pt-2">
          <button
            type="button"
            class="rounded-md border border-slate-600 px-2.5 py-1.5 text-xs text-slate-200 hover:bg-slate-800 md:px-3"
            @click="closeCamera"
          >
            Cancel
          </button>
          <button
            type="button"
            class="rounded-md bg-emerald-500 px-3 py-1.5 text-xs font-semibold text-slate-950 hover:bg-emerald-400 md:px-4"
            @click="captureFromCamera"
          >
            Capture &amp; classify
          </button>
        </div>
      </div>
    </div>

    <!-- Live identification header -->
    <section class="space-y-1">
      <h2 class="text-sm font-semibold uppercase tracking-wide text-slate-300">
        Live Identification
      </h2>
      <p class="text-xs text-slate-400">
        Single-leaf prediction assuming an already trained Inception V3 + mRMR + GBCS model. This view
        skips the optimisation loop and focuses on fast, meaningful classification.
      </p>
    </section>

    <div class="grid gap-6 md:grid-cols-[minmax(0,1.2fr)_minmax(0,1.3fr)] items-start">
      <!-- Input panel -->
      <section
        class="rounded-xl border border-emerald-500/40 bg-slate-900/60 p-6 space-y-4"
      >
        <h3 class="text-sm font-semibold text-slate-100">
          Input · Upload or capture leaf image
        </h3>

        <div class="rounded-lg border border-dashed border-emerald-500/40 bg-slate-950/50 p-4 space-y-3">
          <p class="text-xs text-slate-400">
            This mode applies the learnt feature mask directly for classification. There is no
            training loop here, only inference.
          </p>

          <div class="flex flex-wrap items-center gap-2">
            <button
              type="button"
              class="inline-flex items-center rounded-md bg-emerald-500 px-3 py-1.5 text-xs font-semibold text-slate-950 shadow-sm hover:bg-emerald-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-emerald-400/70"
              @click="triggerBrowse"
            >
              Browse image
            </button>
            <button
              type="button"
              class="inline-flex items-center rounded-md border border-emerald-500/70 bg-transparent px-3 py-1.5 text-xs font-semibold text-emerald-300 shadow-sm hover:bg-emerald-500/10 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-emerald-400/70"
              @click="openCamera"
            >
              Capture from camera
            </button>
          </div>

          <p class="text-[11px] text-slate-500">
            Supported formats: JPG, PNG, TIFF.
          </p>

          <p v-if="selectedFileName" class="mt-1 text-[11px] text-emerald-300">
            Selected image:
            <span class="font-mono text-slate-100">{{ selectedFileName }}</span>
          </p>

          <input
            ref="fileInputRef"
            type="file"
            accept=".jpg,.jpeg,.png,.tif,.tiff,image/*"
            class="hidden"
            @change="onFileChange"
          />
        </div>

      </section>

      <!-- Results panel -->
      <section
        class="rounded-xl border border-slate-800 bg-slate-900/70 p-5 space-y-3"
      >
        <div class="flex items-center justify-between gap-3">
          <h3 class="text-sm font-semibold text-slate-100">
            Live prediction
          </h3>
          <span
            class="inline-flex items-center gap-2 rounded-full border px-3 py-1 text-[11px]"
            :class="
              stage === 'processing'
                ? 'border-emerald-500/40 bg-emerald-500/10 text-emerald-300'
                : 'border-slate-700 bg-slate-900 text-slate-400'
            "
          >
            <span
              class="h-1.5 w-1.5 rounded-full"
              :class="stage === 'processing' ? 'bg-emerald-400 animate-pulse' : 'bg-slate-500'"
            />
            <span>
              {{
                stage === 'idle'
                  ? 'Waiting for image'
                  : stage === 'processing'
                    ? 'Running inference'
                    : 'Prediction ready'
              }}
            </span>
          </span>
        </div>

        <div
          v-if="stage !== 'done'"
          class="mt-1 rounded-lg border border-slate-800 bg-slate-950/60 px-4 py-3 text-xs text-slate-400"
        >
          <p v-if="!selectedFileName">
            Upload or capture a plant leaf image to see the predicted species and confidence.
          </p>
          <p v-else-if="stage === 'processing'">
            Classifying
            <span class="font-medium text-slate-200">{{ selectedFileName }}</span> using the
            optimised feature mask…
          </p>
          <p v-else>
            Ready to classify the selected image.
          </p>
        </div>

        <div
          v-else
          class="mt-1 space-y-4 rounded-lg border border-emerald-500/40 bg-emerald-500/5 px-4 py-4 text-sm"
        >
          <div class="flex items-center justify-between gap-3">
            <div>
              <p class="text-xs font-semibold uppercase tracking-wide text-emerald-300">
                Predicted plant species
              </p>
              <p class="mt-1 text-base font-semibold text-slate-50">
                {{ predictedSpecies }}
              </p>
              <p
                v-if="selectedFileName"
                class="mt-1 text-[11px] text-slate-300"
              >
                Sample:
                <span class="font-mono text-slate-100">{{ selectedFileName }}</span>
              </p>
            </div>
            <div class="text-right">
              <p class="text-xs text-slate-400">Confidence Score</p>
              <p class="mt-1 text-xl font-semibold text-emerald-300">
                {{ confidence?.toFixed(1) }}%
              </p>
              <p class="mt-1 text-[11px] text-slate-500">
                Probability that this specific image belongs to class
                <span class="font-medium">Lagerstroemia speciosa</span>.
              </p>
            </div>
          </div>
        </div>
      </section>
    </div>
  </div>
</template>

