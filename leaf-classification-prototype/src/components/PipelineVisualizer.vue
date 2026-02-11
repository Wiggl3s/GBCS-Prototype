<script setup lang="ts">
import { ref, computed } from 'vue'

type Stage = 'idle' | 'uploading' | 'extracting' | 'optimizing' | 'done'
type DatasetKey = 'swedish' | 'flavia' | 'philippine'

const DATASETS: Array<{
  key: DatasetKey
  label: string
  subtitle: string
  samples: number
  classes: number
  mockAccuracy: number
  mockSelectedFeatures: number
}> = [
  {
    key: 'swedish',
    label: 'Swedish Leaf',
    subtitle: 'Automatic Swedish leaf dataset',
    samples: 1125,
    classes: 15,
    mockAccuracy: 96.8,
    mockSelectedFeatures: 45,
  },
  {
    key: 'flavia',
    label: 'Flavia',
    subtitle: 'Flavia leaf dataset',
    samples: 1907,
    classes: 32,
    mockAccuracy: 95.1,
    mockSelectedFeatures: 52,
  },
  {
    key: 'philippine',
    label: 'Philippine Medicinal',
    subtitle: 'Local medicinal plant leaves',
    samples: 1500,
    classes: 25,
    mockAccuracy: 94.3,
    mockSelectedFeatures: 40,
  },
]

const isDatasetDragging = ref(false)
const selectedDatasetKey = ref<DatasetKey>('swedish')
const selectedFileName = ref<string | null>(null)
const datasetFileNames = ref<Record<DatasetKey, string | null>>({
  swedish: null,
  flavia: null,
  philippine: null,
})
const datasetSampleNames = ref<Record<DatasetKey, string[]>>({
  swedish: [],
  flavia: [],
  philippine: [],
})
const stage = ref<Stage>('idle')
const processing = computed(() => stage.value !== 'idle' && stage.value !== 'done')

const selectedDataset = computed<
  (typeof DATASETS)[number]
>(() => (DATASETS.find((d) => d.key === selectedDatasetKey.value) ?? DATASETS[0]) as (typeof DATASETS)[number])

// mock result state
const predictedSpecies = ref<string | null>(null)
const confidence = ref<number | null>(null)
const featureBefore = ref<number | null>(null)
const featureAfter = ref<number | null>(null)

type RunSummary = {
  id: number
  datasetLabel: string
  sampleName: string | null
  species: string | null
  confidence: number | null
  before: number | null
  after: number | null
}

const runCounter = ref(0)
const previousRuns = ref<RunSummary[]>([])

const datasetInputRef = ref<HTMLInputElement | null>(null)

// comparison & visualization state
const compareBaseline = ref(false)
const optimizationProgress = ref(0)

function formatPathLabel(raw: string): string {
  // Show only the last path segment for readability (handles / or \ separators)
  const parts = raw.split(/[/\\]/)
  return parts[parts.length - 1] || raw
}

function inferDatasetKeyFromPath(raw: string): DatasetKey | null {
  const lower = raw.toLowerCase()
  if (lower.includes('swedish')) return 'swedish'
  if (lower.includes('flavia')) return 'flavia'
  if (lower.includes('philippine') || lower.includes('medicinal')) return 'philippine'
  return null
}

function resetResults() {
  predictedSpecies.value = null
  confidence.value = null
  featureBefore.value = null
  featureAfter.value = null
  optimizationProgress.value = 0
}

function handleDatasetFiles(files: FileList | null) {
  if (!files || files.length === 0) return

  const fileArray = Array.from(files)
  if (!fileArray.length) return

  // Try to infer which dataset this folder corresponds to based on its path/name
  const firstRawPath = (() => {
    const first = fileArray[0]
    if (!first) return ''
    const anyFile = first as any
    if (typeof anyFile?.webkitRelativePath === 'string' && anyFile.webkitRelativePath.length > 0) {
      return anyFile.webkitRelativePath as string
    }
    return first.name
  })()
  const inferredKey = inferDatasetKeyFromPath(firstRawPath)
  const currentKey = inferredKey ?? selectedDatasetKey.value

  // If we could infer it, switch the visible dataset selection
  if (inferredKey) {
    selectedDatasetKey.value = inferredKey
  }

  // Prefer image files inside the folder
  const imageFiles = fileArray.filter((f) => {
    const mime = (f.type ?? '').toLowerCase()
    const name = f.name.toLowerCase()
    const isMimeImage = mime.startsWith('image/')
    const isKnownExt =
      name.endsWith('.jpg') ||
      name.endsWith('.jpeg') ||
      name.endsWith('.png') ||
      name.endsWith('.tif') ||
      name.endsWith('.tiff')
    return isMimeImage || isKnownExt
  })
  const chosen = imageFiles.length > 0 ? imageFiles : fileArray

  const samples = chosen.slice(0, 3)
  datasetSampleNames.value[currentKey] = samples.map((f) => {
    const anyFile = f as any
    const raw =
      typeof anyFile.webkitRelativePath === 'string' && anyFile.webkitRelativePath.length > 0
        ? anyFile.webkitRelativePath
        : f.name
    return formatPathLabel(raw)
  })

  const first = samples[0]
  if (first) {
    const anyFirst = first as any
    const rawLabel =
      (typeof anyFirst.webkitRelativePath === 'string' && anyFirst.webkitRelativePath.length > 0
        ? anyFirst.webkitRelativePath
        : first.name) as string
    datasetFileNames.value[currentKey] = formatPathLabel(rawLabel)
  }
}

function onDatasetDrop(e: DragEvent) {
  e.preventDefault()
  isDatasetDragging.value = false
  handleDatasetFiles(e.dataTransfer?.files ?? null)
}

function onDatasetChange(e: Event) {
  const target = e.target as HTMLInputElement
  handleDatasetFiles(target.files)
}

function triggerDatasetBrowse() {
  datasetInputRef.value?.click()
}

function runSampleFromDataset(name: string) {
  selectedFileName.value = formatPathLabel(name)
  resetResults()
  runMockPipeline()
}

// Note: camera and single-image capture are handled in the LiveIdentification view.

function runMockPipeline() {
  // total simulated pipeline (~6–7s including optimization curve)
  stage.value = 'uploading'

  setTimeout(() => {
    stage.value = 'extracting' // Inception V3
    setTimeout(() => {
      stage.value = 'optimizing' // mRMR + GBCS

      // animate optimization progress over 5 seconds
      optimizationProgress.value = 0
      const duration = 5000
      const start = performance.now()

      const step = (now: number) => {
        const elapsed = now - start
        optimizationProgress.value = Math.min(1, elapsed / duration)
        if (elapsed < duration && stage.value === 'optimizing') {
          requestAnimationFrame(step)
        }
      }

      requestAnimationFrame(step)

      setTimeout(() => {
        // mock results
        stage.value = 'done'
        featureBefore.value = 2048

        // simple per-dataset mock behaviour
        const ds = selectedDataset.value
        predictedSpecies.value =
          ds.key === 'swedish'
            ? 'Quercus robur (English Oak)'
            : ds.key === 'flavia'
              ? 'Acer palmatum (Japanese Maple)'
              : 'Vitex negundo (Lagundi)'

        confidence.value = ds.mockAccuracy
        featureAfter.value = ds.mockSelectedFeatures

        // push into comparison history (keep last 4 runs)
        const id = ++runCounter.value
        const summary: RunSummary = {
          id,
          datasetLabel: ds.label,
          sampleName: selectedFileName.value,
          species: predictedSpecies.value,
          confidence: confidence.value,
          before: featureBefore.value,
          after: featureAfter.value,
        }
        previousRuns.value = [summary, ...previousRuns.value].slice(0, 4)
      }, 900) // GBCS
    }, 1100) // Inception V3
  }, 500) // initial upload / validation
}

const stageLabel = computed(() => {
  switch (stage.value) {
    case 'uploading':
      return 'Preparing image…'
    case 'extracting':
      return 'Inception V3 Processing'
    case 'optimizing':
      return 'Running GBCS Optimization'
    case 'done':
      return 'Pipeline complete'
    default:
      return 'Waiting for image input'
  }
})

const stageStep = computed(() => {
  switch (stage.value) {
    case 'uploading':
      return 0
    case 'extracting':
      return 1
    case 'optimizing':
      return 2
    case 'done':
      return 3
    default:
      return 0
  }
})
</script>

<template>
  <div class="w-full max-w-5xl mx-auto space-y-8">
    <!-- Title / context -->
    <header class="flex flex-col gap-4 md:flex-row md:items-end md:justify-between">
      <div class="space-y-1">
        <p class="text-xs font-semibold uppercase tracking-[0.2em] text-emerald-400">
          GBCS · Prototype
        </p>
        <h2 class="text-2xl font-semibold text-slate-50">
            Optimized GBCS for Plant Leaf Classification
        </h2>
        <p class="text-sm text-slate-400 max-w-2xl">
          Simulated flow from image acquisition through Inception V3 feature extraction and
          GBCS‑based feature selection. Choose a dataset to see how the optimizer behaves.
        </p>
      </div>

      <!-- Dataset selector -->
      <div class="rounded-xl border border-slate-800 bg-slate-900/70 p-3 text-xs space-y-2">
        <p class="text-[11px] font-semibold text-slate-300 uppercase tracking-wide">
          Dataset
        </p>
        <div class="flex flex-wrap gap-2">
          <button
            v-for="ds in DATASETS"
            :key="ds.key"
            type="button"
            class="inline-flex items-center gap-2 rounded-full border px-3 py-1.5 transition text-[11px]"
            :class="
              ds.key === selectedDatasetKey
                ? 'border-emerald-500/60 bg-emerald-500/10 text-emerald-200'
                : 'border-slate-700 bg-slate-900 text-slate-300 hover:border-slate-500'
            "
            @click="selectedDatasetKey = ds.key"
          >
            <span
              class="h-1.5 w-1.5 rounded-full"
              :class="
                ds.key === selectedDatasetKey ? 'bg-emerald-400' : 'bg-slate-500'
              "
            />
            <span>{{ ds.label }}</span>
          </button>
        </div>
        <p class="text-[11px] text-slate-400">
          Selected: <span class="font-medium text-slate-200">{{ selectedDataset.label }}</span>
          · {{ selectedDataset.classes }} classes, {{ selectedDataset.samples }} images
        </p>
      </div>
    </header>

    <!-- Layout: upload · pipeline · results -->
    <div class="grid gap-6 lg:grid-cols-[minmax(0,1.4fr)_minmax(0,1.2fr)]">
      <div class="space-y-6">
        <!-- Dataset upload (full dataset file) -->
        <section
          class="rounded-xl border border-dashed border-slate-700 bg-slate-900/60 p-5 transition-colors"
          :class="{
            'bg-sky-500/10 border-sky-400': isDatasetDragging,
          }"
          @dragover.prevent="isDatasetDragging = true"
          @dragleave.prevent="isDatasetDragging = false"
          @drop="onDatasetDrop"
        >
          <div class="flex flex-col gap-3">
            <div class="flex items-center justify-between gap-3">
              <div>
                <h3 class="text-sm font-semibold text-slate-100">
                  Dataset file · {{ selectedDataset.label }}
                </h3>
                <p class="text-xs text-slate-400">
                  Attach the feature matrix or metadata file (e.g., CSV/NPY) for this dataset to
                  enable global metric comparison.
                </p>
              </div>
              <span class="text-[11px] text-slate-500">
                Per‑dataset upload
              </span>
            </div>

            <div class="flex flex-wrap items-center gap-2">
              <button
                type="button"
                class="inline-flex items-center rounded-md bg-sky-500 px-3 py-1.5 text-xs font-semibold text-slate-950 shadow-sm hover:bg-sky-400 focus-visible:outline-none focus-visible:ring-2 focus-visible:ring-sky-400/70"
                @click="triggerDatasetBrowse"
              >
                Browse dataset folder
              </button>
              <p class="text-[11px] text-slate-500">
                or drop a folder into this area
              </p>
            </div>

            <p class="text-[11px] text-slate-400">
              Current folder / file:
              <span v-if="datasetFileNames[selectedDatasetKey]" class="font-medium text-slate-100">
                {{ datasetFileNames[selectedDatasetKey] }}
              </span>
              <span v-else class="italic text-slate-500">
                none uploaded yet
              </span>
            </p>

            <input
              ref="datasetInputRef"
              type="file"
              webkitdirectory
              class="hidden"
              @change="onDatasetChange"
            />

            <div
              v-if="datasetSampleNames[selectedDatasetKey]?.length"
              class="mt-3 rounded-md border border-slate-800 bg-slate-950/40 px-3 py-2 text-[11px] space-y-1"
            >
              <p class="text-slate-400">
                Sample images detected in this folder (click to simulate the pipeline):
              </p>
              <div class="flex flex-wrap gap-2">
                <button
                  v-for="sample in datasetSampleNames[selectedDatasetKey]"
                  :key="sample"
                  type="button"
                  class="rounded-full border border-slate-700 bg-slate-900 px-3 py-1 hover:border-emerald-400 hover:text-emerald-200 transition"
                  @click="runSampleFromDataset(sample)"
                >
                  {{ sample }}
                </button>
              </div>
            </div>
          </div>
        </section>

        <!-- Processing visualization -->
        <section
          class="rounded-xl border border-slate-800 bg-slate-900/60 p-5 space-y-4"
        >
          <div class="flex items-center justify-between gap-3">
            <div>
              <h3 class="text-sm font-semibold text-slate-100">
                Processing pipeline
              </h3>
              <p class="text-xs text-slate-400">
                Status: {{ stageLabel }}
              </p>
            </div>
            <span
              class="inline-flex items-center gap-2 rounded-full border px-3 py-1 text-[11px]"
              :class="
                processing
                  ? 'border-emerald-500/40 bg-emerald-500/10 text-emerald-300'
                  : 'border-slate-700 bg-slate-900 text-slate-400'
              "
            >
              <span
                class="h-1.5 w-1.5 rounded-full"
                :class="processing ? 'bg-emerald-400 animate-pulse' : 'bg-slate-500'"
              />
              <span>{{ processing ? 'Simulating backend' : 'Idle' }}</span>
            </span>
          </div>

          <!-- Stepper -->
          <ol class="space-y-3 text-xs text-slate-300">
            <li class="flex items-start gap-3">
              <div
                class="mt-0.5 h-5 w-5 shrink-0 rounded-full border flex items-center justify-center"
                :class="
                  stageStep >= 1
                    ? 'border-emerald-400 bg-emerald-500/10 text-emerald-300'
                    : 'border-slate-700 text-slate-500'
                "
              >
                1
              </div>
              <div>
                <p class="font-semibold">
                  Inception V3 · 2048-D feature vector
                </p>
                <p class="text-[11px] text-slate-400">
                  CNN backbone encodes the leaf image into a 2048‑dimensional descriptor.
                </p>
              </div>
            </li>

            <li class="flex items-start gap-3">
              <div
                class="mt-0.5 h-5 w-5 shrink-0 rounded-full border flex items-center justify-center"
                :class="
                  stageStep >= 2
                    ? 'border-emerald-400 bg-emerald-500/10 text-emerald-300'
                    : 'border-slate-700 text-slate-500'
                "
              >
                2
              </div>
              <div>
                <p class="font-semibold">
                  mRMR Filter · top 500 features
                </p>
                <p class="text-[11px] text-slate-400">
                  Minimum Redundancy Maximum Relevance filter prunes redundant and noisy dimensions.
                </p>
              </div>
            </li>

            <li class="flex items-start gap-3">
              <div
                class="mt-0.5 h-5 w-5 shrink-0 rounded-full border flex items-center justify-center"
                :class="
                  stageStep >= 3
                    ? 'border-emerald-400 bg-emerald-500/10 text-emerald-300'
                    : 'border-slate-700 text-slate-500'
                "
              >
                3
              </div>
              <div>
                <p class="font-semibold">
                  GBCS · final subset &amp; classification
                </p>
                <p class="text-[11px] text-slate-400">
                  Genetic Binary Cuckoo Search refines a compact subset (e.g. 120 dims) used by the classifier and
                  exposed in the metrics.
                </p>
              </div>
            </li>
          </ol>

          <!-- Optimization convergence placeholder -->
          <div class="mt-4 rounded-lg border border-slate-800 bg-slate-950/60 px-3 py-2">
            <p class="mb-2 text-[11px] font-semibold text-slate-300">
              Optimization convergence (mock) · fitness vs. iterations
            </p>
            <div class="relative h-28 w-full">
              <svg viewBox="0 0 100 50" class="h-full w-full text-emerald-400">
                <defs>
                  <linearGradient id="fitness-line" x1="0" y1="0" x2="1" y2="0">
                    <stop offset="0%" stop-color="#22c55e" />
                    <stop offset="100%" stop-color="#a3e635" />
                  </linearGradient>
                </defs>
                <polyline
                  fill="none"
                  stroke="url(#fitness-line)"
                  stroke-width="1.5"
                  :stroke-dasharray="100"
                  :stroke-dashoffset="100 - 100 * optimizationProgress"
                  points="0,45 15,38 30,32 50,24 75,15 100,8"
                />
                <line x1="0" y1="45" x2="100" y2="45" stroke="#1e293b" stroke-width="0.5" />
                <line x1="0" y1="5" x2="0" y2="45" stroke="#1e293b" stroke-width="0.5" />
              </svg>
              <div
                class="pointer-events-none absolute inset-0 flex items-start justify-between px-1 text-[9px] text-slate-500"
              >
                <span>0</span>
                <span>25</span>
                <span>50</span>
              </div>
              <div
                class="pointer-events-none absolute inset-y-0 right-1 flex flex-col justify-between py-1 text-[9px] text-slate-500"
              >
                <span>0.95</span>
                <span>0.60</span>
              </div>
            </div>
            <p class="mt-1 text-[10px] text-slate-500">
              Placeholder curve rising from 0.60 to 0.95 while the combined mRMR + GBCS optimization stage is running.
            </p>
          </div>
        </section>
      </div>

      <!-- Results card -->
      <section
        class="h-full rounded-xl border border-slate-800 bg-slate-900/70 p-5 flex flex-col justify-between"
      >
        <div class="space-y-3">
          <div class="flex items-center justify-between gap-3">
            <h3 class="text-sm font-semibold text-slate-100">
              Results interface
            </h3>
            <div class="flex items-center gap-4 text-[11px]">
              <label class="inline-flex items-center gap-2 text-slate-400">
                <span>Compare vs. Baseline BCS</span>
                <button
                  type="button"
                  class="relative inline-flex h-4 w-7 items-center rounded-full border border-slate-600 transition"
                  :class="compareBaseline ? 'bg-emerald-500/40 border-emerald-400' : 'bg-slate-900'"
                  @click="compareBaseline = !compareBaseline"
                >
                  <span
                    class="inline-block h-3 w-3 rounded-full bg-slate-200 transition"
                    :class="compareBaseline ? 'translate-x-3.5' : 'translate-x-0.5'"
                  />
                </button>
              </label>
              <span class="text-slate-500">
                Mocked · no backend connection
              </span>
            </div>
          </div>

          <div class="rounded-lg border border-slate-800 bg-slate-950/40 px-3 py-2 text-[11px] space-y-1">
            <p class="text-slate-400">
              Dataset:
              <span class="font-medium text-slate-100">{{ selectedDataset.label }}</span>
              · {{ selectedDataset.classes }} classes,
              {{ selectedDataset.samples }} images
            </p>
            <p class="text-slate-500">
              Dataset file:
              <span
                v-if="datasetFileNames[selectedDatasetKey]"
                class="font-medium text-slate-200"
              >
                {{ datasetFileNames[selectedDatasetKey] }}
              </span>
              <span v-else class="italic text-slate-500">not uploaded</span>
            </p>
            <p v-if="datasetSampleNames[selectedDatasetKey]?.length" class="text-slate-500">
              Using <span class="font-medium text-slate-200">{{ datasetSampleNames[selectedDatasetKey].length }}</span>
              sample image<span v-if="datasetSampleNames[selectedDatasetKey].length > 1">s</span> from the folder for
              quick inspection.
            </p>
          </div>

          <div
            v-if="stage !== 'done'"
            class="mt-3 rounded-lg border border-slate-800 bg-slate-950/60 px-4 py-3 text-xs text-slate-400"
          >
            <p v-if="!selectedFileName">
              Upload a leaf image to simulate the classification pipeline.
            </p>
            <p v-else-if="processing">
              Processing <span class="font-medium text-slate-200">{{ selectedFileName }}</span>…
              This mock run takes about 2–3 seconds.
            </p>
            <p v-else>
              Ready for next image.
            </p>
          </div>

          <div
            v-else
            class="mt-3 space-y-4 rounded-lg border border-emerald-500/40 bg-emerald-500/5 px-4 py-4 text-sm"
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
                <p class="text-xs text-slate-400">Model Accuracy</p>
                <p class="mt-1 text-xl font-semibold text-emerald-300">
                  95.1%
                </p>
                <p class="mt-1 text-[11px] text-slate-500">
                  Validated on 1,907 images.
                </p>
              </div>
            </div>

            <div
              class="grid grid-cols-2 gap-3 rounded-md border border-emerald-500/20 bg-slate-950/40 px-3 py-2 text-xs"
            >
              <div>
                <p class="text-slate-400">Feature dimensionality</p>
                <p class="mt-1 font-mono text-[12px] text-slate-100">
                  {{ featureBefore }} → <span class="text-emerald-300">{{ featureAfter }}</span>
                </p>
              </div>
              <div>
                <p class="text-slate-400">Optimization summary</p>
                <p class="mt-1 text-[12px] text-slate-200">
                  GBCS reduced features from {{ featureBefore }} to {{ featureAfter }}.
                </p>
              </div>
            </div>

            <!-- Baseline comparison table -->
            <div
              v-if="compareBaseline"
              class="mt-3 rounded-md border border-emerald-500/20 bg-slate-950/60 px-3 py-2 text-[11px]"
            >
              <p class="mb-2 font-semibold text-slate-200">
                Comparison vs. baseline Binary Cuckoo Search (BCS)
              </p>
              <div class="grid grid-cols-3 gap-2 text-center">
                <div class="space-y-1">
                  <p class="text-slate-500">Metric</p>
                  <p class="font-medium text-slate-300">Iterations</p>
                  <p class="font-medium text-slate-300">Selected features</p>
                </div>
                <div class="space-y-1">
                  <p class="text-emerald-400 font-semibold">GBCS</p>
                  <p>50</p>
                  <p>120</p>
                </div>
                <div class="space-y-1">
                  <p class="text-slate-400 font-semibold">Baseline BCS</p>
                  <p>85</p>
                  <p>≈ 200</p>
                </div>
              </div>
            </div>

            <div class="pt-1 border-t border-emerald-500/10 mt-3 text-[11px] text-slate-400">
              <p>
                In this mock comparison, GBCS achieves
                <span class="font-medium text-emerald-300">
                  {{ selectedDataset.mockAccuracy.toFixed(1) }}% accuracy
                </span>
                on {{ selectedDataset.label }} with
                <span class="font-medium text-emerald-300">
                  {{ selectedDataset.mockSelectedFeatures }}
                </span>
                selected features.
              </p>
            </div>
          </div>

          <!-- Comparison history -->
          <div
            v-if="previousRuns.length > 1"
            class="mt-5 rounded-lg border border-slate-800 bg-slate-950/60 px-4 py-3 text-[11px] space-y-3"
          >
            <p class="font-semibold text-slate-300">
              Recent runs (for comparison)
            </p>
            <div class="space-y-2">
              <div
                v-for="run in previousRuns"
                :key="run.id"
                class="flex flex-wrap items-center justify-between gap-3 rounded-md border border-slate-800/80 bg-slate-950/60 px-3 py-2"
              >
                <div class="space-y-1 min-w-0">
                  <p class="text-slate-200 truncate">
                    {{ run.species }}<span class="text-slate-500"> · {{ run.datasetLabel }}</span>
                  </p>
                  <p class="text-slate-400 truncate">
                    Sample:
                    <span class="font-mono text-slate-100">
                      {{ run.sampleName ?? '—' }}
                    </span>
                  </p>
                </div>
                <div class="text-right space-y-1">
                  <p class="text-emerald-300 font-semibold">
                    {{ run.confidence?.toFixed(1) ?? '—' }}%
                  </p>
                  <p class="text-slate-400">
                    {{ run.before ?? '—' }} →
                    <span class="text-emerald-300">{{ run.after ?? '—' }}</span>
                  </p>
                </div>
              </div>
            </div>
          </div>

        </div>
      </section>
    </div>
  </div>
</template>