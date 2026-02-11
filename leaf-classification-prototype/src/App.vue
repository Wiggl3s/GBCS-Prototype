<script setup lang="ts">
import { ref } from 'vue'
import BenchmarkTraining from './components/PipelineVisualizer.vue'
import LiveIdentification from './components/LiveIdentification.vue'

type TabKey = 'benchmark' | 'live'

const activeTab = ref<TabKey>('benchmark')

const tabs: { key: TabKey; label: string; subtitle: string }[] = [
  {
    key: 'benchmark',
    label: 'Benchmark & Training',
    subtitle: 'Dataset-level evaluation, optimization loop, and feature reduction',
  },
  {
    key: 'live',
    label: 'Live Identification',
    subtitle: 'Single-leaf prediction using pre-optimized feature mask',
  },
]
</script>

<template>
  <main class="w-full max-w-5xl mx-auto px-6 py-8 space-y-6">
    <header class="space-y-3">
      

      <!-- Simple tabbed navbar -->
      <nav class="flex flex-wrap items-center gap-2 text-sm">
        <button
          v-for="tab in tabs"
          :key="tab.key"
          type="button"
          class="rounded-full border px-3 py-1.5 transition text-left"
          :class="
            tab.key === activeTab
              ? 'border-emerald-500/70 bg-emerald-500/10 text-emerald-200'
              : 'border-slate-700 bg-slate-900 text-slate-300 hover:border-slate-500'
          "
          @click="activeTab = tab.key"
        >
          <div class="font-semibold">
            {{ tab.label }}
          </div>
          <div class="text-[11px] text-slate-400">
            {{ tab.subtitle }}
          </div>
        </button>
      </nav>
    </header>

    <section v-if="activeTab === 'benchmark'">
      <BenchmarkTraining />
    </section>
    <section v-else>
      <LiveIdentification />
    </section>
  </main>
</template>
