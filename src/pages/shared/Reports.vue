<script setup>
import { ref, computed, watch, onMounted, onBeforeUnmount } from 'vue'
import DashboardLayout from '@/layouts/DashboardLayout.vue'
import BaseButton from '@/components/ui/BaseButton.vue'
import BaseBadge from '@/components/ui/BaseBadge.vue'
import SkeletonCard from '@/components/skeletons/SkeletonCard.vue'
import { useAuthStore } from '@/stores/auth'
import { useUiStore } from '@/stores/ui'
import {
  ArrowDownTrayIcon,
  ArrowTopRightOnSquareIcon,
  BeakerIcon,
  BuildingOffice2Icon,
  DocumentCheckIcon,
  DocumentTextIcon,
  EyeIcon,
  ExclamationTriangleIcon,
  HeartIcon,
  InboxArrowDownIcon,
  MagnifyingGlassIcon,
  PaperAirplaneIcon,
  XMarkIcon,
} from '@heroicons/vue/24/outline'

const auth = useAuthStore()
const ui = useUiStore()
const isAdmin = computed(() => auth.role === 'super-admin')

const loading = ref(true)
onMounted(() => setTimeout(() => (loading.value = false), 600))

/* ---------- data ---------- */
// Demo: sob entry ekta sample PDF e point kore. Real file URL pele (API/storage) shudhu `file` ta bodle dao.
const SAMPLE_PDF = '/documents/sample.pdf'

const documents = [
  { id: 'doc_01', title: 'Data processing agreement', party: 'Meridian General Hospital', direction: 'sent', date: '2026-10-28', size: '320 KB', file: SAMPLE_PDF },
  { id: 'doc_02', title: 'Onboarding and integration guide', party: 'CarePlus Clinic', direction: 'sent', date: '2026-10-20', size: '1.4 MB', file: SAMPLE_PDF },
  { id: 'doc_03', title: 'API access terms', party: 'GreenLeaf Pharmacy', direction: 'sent', date: '2026-09-30', size: '180 KB', file: SAMPLE_PDF },
  { id: 'doc_04', title: 'Security compliance summary', party: 'Meridian Central Pharmacy', direction: 'sent', date: '2026-09-12', size: '540 KB', file: SAMPLE_PDF },
  { id: 'doc_05', title: 'Hospital operating licence', party: 'Meridian General Hospital', direction: 'received', date: '2026-10-25', size: '860 KB', file: SAMPLE_PDF },
  { id: 'doc_06', title: 'Clinic registration certificate', party: 'CarePlus Clinic', direction: 'received', date: '2026-10-18', size: '610 KB', file: SAMPLE_PDF },
  { id: 'doc_07', title: 'Pharmacy trade licence', party: 'GreenLeaf Pharmacy', direction: 'received', date: '2026-09-28', size: '450 KB', file: SAMPLE_PDF },
  { id: 'doc_08', title: 'Insurance certificate', party: 'Meridian Central Pharmacy', direction: 'received', date: '2026-09-05', size: '390 KB', file: SAMPLE_PDF },
  { id: 'doc_09', title: 'Drug licence renewal', party: 'Wellness Point Pharmacy', direction: 'received', date: '2026-07-22', size: '280 KB', file: SAMPLE_PDF },
]

const contractsRaw = [
  { id: 'con_01', facility: 'Meridian General Hospital', type: 'Hospital', status: 'active', start: '2025-01-15', end: '2027-06-30', file: SAMPLE_PDF },
  { id: 'con_02', facility: 'Meridian Central Pharmacy', type: 'Pharmacy', status: 'active', start: '2025-03-01', end: '2026-12-31', file: SAMPLE_PDF },
  { id: 'con_03', facility: 'GreenLeaf Pharmacy', type: 'Pharmacy', status: 'active', start: '2025-06-10', end: '2027-06-09', file: SAMPLE_PDF },
  { id: 'con_04', facility: 'CarePlus Clinic', type: 'Clinic', status: 'pending', start: '2026-10-20', end: '2027-10-19', file: SAMPLE_PDF },
  { id: 'con_05', facility: 'Wellness Point Pharmacy', type: 'Pharmacy', status: 'expired', start: '2024-08-01', end: '2026-07-31', file: SAMPLE_PDF },
]

const reports = [
  { id: 1, name: 'Monthly adherence summary', period: 'July 2026', size: '212 KB', type: 'PDF' },
  { id: 2, name: 'Prescription activity log', period: 'Q2 2026', size: '480 KB', type: 'CSV' },
  { id: 3, name: 'Patient outcomes report', period: 'H1 2026', size: '1.1 MB', type: 'PDF' },
]

const typeIcon = { Hospital: BuildingOffice2Icon, Clinic: HeartIcon, Pharmacy: BeakerIcon }
const fmtDate = (d) => new Date(d).toLocaleDateString('en-GB', { day: 'numeric', month: 'short', year: 'numeric' })
const daysLeft = (d) => Math.ceil((new Date(d) - new Date()) / 86400000)

/* ---------- contracts (status derived from end date) ---------- */
const contracts = contractsRaw.map((c) => {
  const left = daysLeft(c.end)
  let state = c.status
  if (c.status === 'active' && left < 0) state = 'expired'
  else if (c.status === 'active' && left <= 60) state = 'expiring'
  return { ...c, state, left }
})
const contractTone = { active: 'success', expiring: 'warning', pending: 'warning', expired: 'danger' }
const contractLabel = { active: 'Active', expiring: 'Expiring soon', pending: 'Pending signature', expired: 'Expired' }

const inForce = (c) => c.state === 'active' || c.state === 'expiring'
const contractMatches = (key, c) => (key === 'all' ? true : key === 'active' ? inForce(c) : c.state === key)

const contractFilter = ref('all')
const contractChips = computed(() =>
  [
    { key: 'all', label: 'All' },
    { key: 'active', label: 'Active' },
    { key: 'expiring', label: 'Expiring soon' },
    { key: 'pending', label: 'Pending' },
    { key: 'expired', label: 'Expired' },
  ].map((c) => ({ ...c, count: contracts.filter((x) => contractMatches(c.key, x)).length }))
)
const filteredContracts = computed(() => contracts.filter((c) => contractMatches(contractFilter.value, c)))

/* ---------- documents filters ---------- */
const docDir = ref('all')
const docSearch = ref('')
const sentCount = documents.filter((d) => d.direction === 'sent').length
const receivedCount = documents.filter((d) => d.direction === 'received').length

const docChips = [
  { key: 'all', label: 'All', count: documents.length },
  { key: 'sent', label: 'Sent', count: sentCount },
  { key: 'received', label: 'Received', count: receivedCount },
]
const filteredDocs = computed(() => {
  const q = docSearch.value.trim().toLowerCase()
  return documents.filter(
    (d) =>
      (docDir.value === 'all' || d.direction === docDir.value) &&
      (!q || d.title.toLowerCase().includes(q) || d.party.toLowerCase().includes(q))
  )
})

/* ---------- tabs + summary tiles ---------- */
const tab = ref(isAdmin.value ? 'documents' : 'reports')
const tabs = computed(() =>
  isAdmin.value
    ? [
        { key: 'documents', label: 'Documents', count: documents.length },
        { key: 'contracts', label: 'Contracts', count: contracts.length },
        { key: 'reports', label: 'Reports', count: reports.length },
      ]
    : [{ key: 'reports', label: 'Reports', count: reports.length }]
)

const activeContracts = contracts.filter(inForce).length
const expiringCount = contracts.filter((c) => c.state === 'expiring').length

const tiles = [
  { key: 'sent', label: 'Documents sent', value: sentCount, icon: PaperAirplaneIcon, go: () => { tab.value = 'documents'; docDir.value = 'sent' } },
  { key: 'received', label: 'Documents received', value: receivedCount, icon: InboxArrowDownIcon, go: () => { tab.value = 'documents'; docDir.value = 'received' } },
  { key: 'active', label: 'Active contracts', value: activeContracts, icon: DocumentCheckIcon, go: () => { tab.value = 'contracts'; contractFilter.value = 'active' } },
  { key: 'expiring', label: 'Expiring soon', value: expiringCount, icon: ExclamationTriangleIcon, accent: true, go: () => { tab.value = 'contracts'; contractFilter.value = 'expiring' } },
]

/* ---------- PDF viewer ---------- */
const viewing = ref(null)
function openViewer(title, subtitle, file) {
  viewing.value = { title, subtitle, file }
}
function closeViewer() {
  viewing.value = null
}
const onKey = (e) => e.key === 'Escape' && closeViewer()
watch(viewing, (open) => {
  if (open) window.addEventListener('keydown', onKey)
  else window.removeEventListener('keydown', onKey)
})
onBeforeUnmount(() => window.removeEventListener('keydown', onKey))

/* ---------- reports (unchanged) ---------- */
function download(r) {
  ui.toast({ type: 'success', title: 'Export started', message: `Preparing ${r.name}.${r.type.toLowerCase()}` })
}

const inputCls =
  'w-full rounded-lg border border-meridian-200 dark:border-white/10 bg-white dark:bg-white/5 py-2 pl-9 pr-3 text-sm text-meridian-800 dark:text-meridian-100 placeholder:text-meridian-400 focus:border-meridian-500 focus:outline-none focus:ring-2 focus:ring-meridian-500/20'
const iconBtn =
  'inline-flex items-center justify-center rounded-lg border border-meridian-200 p-2 text-meridian-500 transition hover:text-meridian-700 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:border-white/10'
const viewBtn =
  'inline-flex items-center gap-1.5 rounded-lg border border-meridian-200 px-3 py-2 text-sm font-medium text-meridian-600 transition hover:border-meridian-400 hover:text-meridian-800 focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:border-white/10 dark:text-meridian-100'
</script>

<template>
  <DashboardLayout>
    <!-- Header -->
    <div class="mb-6 flex flex-wrap items-end justify-between gap-4">
      <div>
        <h1 class="font-display text-2xl font-semibold text-meridian-900 dark:text-white">Reports</h1>
        <p v-if="isAdmin" class="mt-1 text-sm text-meridian-500 dark:text-meridian-300">
          Documents exchanged with partners, medical contracts and exported reports.
        </p>
      </div>
      <BaseButton variant="outline"><template #icon-left><ArrowDownTrayIcon class="h-5 w-5" /></template>Export all</BaseButton>
    </div>

    <!-- Summary tiles (click to jump to the matching list) -->
    <div v-if="isAdmin" class="mb-6 grid grid-cols-1 gap-4 sm:grid-cols-2 lg:grid-cols-4">
      <template v-if="loading"><SkeletonCard v-for="i in 4" :key="i" /></template>
      <template v-else>
        <button
          v-for="t in tiles"
          :key="t.key"
          type="button"
          @click="t.go()"
          class="flex items-center gap-4 rounded-2xl border border-meridian-100 bg-white p-4 text-left transition hover:border-meridian-300 hover:shadow-md focus:outline-none focus-visible:ring-2 focus-visible:ring-meridian-500 dark:border-white/10 dark:bg-white/5 dark:hover:border-white/25"
        >
          <span
            :class="[
              'flex h-11 w-11 shrink-0 items-center justify-center rounded-xl',
              t.accent ? 'bg-[#ff6b5b]/10 text-[#ff6b5b]' : 'bg-meridian-50 text-meridian-600 dark:bg-white/10',
            ]"
          >
            <component :is="t.icon" class="h-5 w-5" />
          </span>
          <div>
            <p class="text-2xl font-semibold tabular-nums text-meridian-900 dark:text-white">{{ t.value }}</p>
            <p class="text-sm text-meridian-500 dark:text-meridian-300">{{ t.label }}</p>
          </div>
        </button>
      </template>
    </div>

    <!-- Tabs -->
    <div v-if="isAdmin" class="mb-5 inline-flex rounded-xl border border-meridian-100 bg-white p-1 dark:border-white/10 dark:bg-white/5" role="tablist">
      <button
        v-for="t in tabs"
        :key="t.key"
        type="button"
        role="tab"
        :aria-selected="tab === t.key"
        @click="tab = t.key"
        :class="[
          'inline-flex items-center gap-2 rounded-lg px-4 py-1.5 text-sm font-medium transition',
          tab === t.key
            ? 'bg-meridian-600 text-white shadow-sm'
            : 'text-meridian-600 hover:bg-meridian-50 dark:text-meridian-200 dark:hover:bg-white/10',
        ]"
      >
        {{ t.label }}
        <span :class="['text-xs tabular-nums', tab === t.key ? 'text-white/80' : 'text-meridian-400']">{{ t.count }}</span>
      </button>
    </div>

    <div v-if="loading && isAdmin" class="h-72 animate-pulse rounded-2xl bg-meridian-100 dark:bg-white/5" />

    <template v-else>
      <!-- Documents -->
      <section v-if="tab === 'documents'">
        <div class="mb-4 flex flex-wrap items-center justify-between gap-3">
          <div class="flex flex-wrap items-center gap-2">
            <button
              v-for="c in docChips"
              :key="c.key"
              type="button"
              @click="docDir = c.key"
              :class="[
                'inline-flex items-center gap-1.5 rounded-full border px-3 py-1.5 text-sm font-medium transition',
                docDir === c.key
                  ? 'border-meridian-600 bg-meridian-600 text-white'
                  : 'border-meridian-200 bg-white text-meridian-600 hover:bg-meridian-50 dark:border-white/10 dark:bg-white/5 dark:text-meridian-200 dark:hover:bg-white/10',
              ]"
            >
              {{ c.label }}
              <span :class="['text-xs tabular-nums', docDir === c.key ? 'text-white/80' : 'text-meridian-400']">{{ c.count }}</span>
            </button>
          </div>

          <div class="relative w-full sm:w-72">
            <MagnifyingGlassIcon class="pointer-events-none absolute left-3 top-1/2 h-4 w-4 -translate-y-1/2 text-meridian-400" />
            <input v-model="docSearch" type="search" placeholder="Search documents or partners…" aria-label="Search documents" :class="inputCls" />
          </div>
        </div>

        <ul v-if="filteredDocs.length" class="card-base divide-y divide-meridian-100 dark:divide-white/5">
          <li v-for="d in filteredDocs" :key="d.id" class="flex flex-col gap-3 p-5 sm:flex-row sm:items-center sm:justify-between">
            <div class="flex min-w-0 items-center gap-3">
              <span class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-meridian-50 text-meridian-600 dark:bg-white/5">
                <DocumentTextIcon class="h-5 w-5" />
              </span>
              <div class="min-w-0">
                <p class="truncate text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ d.title }}</p>
                <p class="text-xs text-meridian-500">
                  {{ d.direction === 'sent' ? 'To' : 'From' }} {{ d.party }} · {{ fmtDate(d.date) }} · {{ d.size }}
                </p>
              </div>
            </div>

            <div class="flex items-center gap-2 sm:shrink-0">
              <BaseBadge tone="neutral">{{ d.direction === 'sent' ? 'Sent' : 'Received' }}</BaseBadge>
              <button type="button" :class="viewBtn" @click="openViewer(d.title, `${d.direction === 'sent' ? 'To' : 'From'} ${d.party}`, d.file)">
                <EyeIcon class="h-4 w-4" />
                View PDF
              </button>
              <a :href="d.file" :download="`${d.title}.pdf`" :class="iconBtn" :aria-label="`Download ${d.title}`">
                <ArrowDownTrayIcon class="h-4 w-4" />
              </a>
            </div>
          </li>
        </ul>
        <div v-else class="flex flex-col items-center rounded-2xl border border-dashed border-meridian-200 px-6 py-14 text-center dark:border-white/10">
          <DocumentTextIcon class="h-10 w-10 text-meridian-400" />
          <h2 class="mt-3 font-medium text-meridian-800 dark:text-meridian-100">No documents found</h2>
          <p class="mt-1 text-sm text-meridian-500">Try a different search or switch the filter.</p>
        </div>
      </section>

      <!-- Contracts -->
      <section v-else-if="tab === 'contracts'">
        <div class="mb-4 flex flex-wrap items-center gap-2">
          <button
            v-for="c in contractChips"
            :key="c.key"
            type="button"
            @click="contractFilter = c.key"
            :class="[
              'inline-flex items-center gap-1.5 rounded-full border px-3 py-1.5 text-sm font-medium transition',
              contractFilter === c.key
                ? 'border-meridian-600 bg-meridian-600 text-white'
                : 'border-meridian-200 bg-white text-meridian-600 hover:bg-meridian-50 dark:border-white/10 dark:bg-white/5 dark:text-meridian-200 dark:hover:bg-white/10',
            ]"
          >
            {{ c.label }}
            <span :class="['text-xs tabular-nums', contractFilter === c.key ? 'text-white/80' : 'text-meridian-400']">{{ c.count }}</span>
          </button>
        </div>

        <ul v-if="filteredContracts.length" class="card-base divide-y divide-meridian-100 dark:divide-white/5">
          <li v-for="c in filteredContracts" :key="c.id" class="flex flex-col gap-3 p-5 sm:flex-row sm:items-center sm:justify-between">
            <div class="flex min-w-0 items-center gap-3">
              <span class="flex h-10 w-10 shrink-0 items-center justify-center rounded-xl bg-meridian-50 text-meridian-600 dark:bg-white/5">
                <component :is="typeIcon[c.type]" class="h-5 w-5" />
              </span>
              <div class="min-w-0">
                <p class="truncate text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ c.facility }}</p>
                <p class="text-xs text-meridian-500">
                  {{ c.type }} · {{ fmtDate(c.start) }} – {{ fmtDate(c.end) }}
                  <span v-if="c.state === 'expiring'" class="font-medium text-amber-600"> · Ends in {{ c.left }} days</span>
                </p>
              </div>
            </div>

            <div class="flex items-center gap-2 sm:shrink-0">
              <BaseBadge :tone="contractTone[c.state]">{{ contractLabel[c.state] }}</BaseBadge>
              <button type="button" :class="viewBtn" @click="openViewer(`${c.facility} contract`, `${fmtDate(c.start)} – ${fmtDate(c.end)}`, c.file)">
                <EyeIcon class="h-4 w-4" />
                View PDF
              </button>
              <a :href="c.file" :download="`${c.facility} contract.pdf`" :class="iconBtn" :aria-label="`Download ${c.facility} contract`">
                <ArrowDownTrayIcon class="h-4 w-4" />
              </a>
            </div>
          </li>
        </ul>
        <div v-else class="flex flex-col items-center rounded-2xl border border-dashed border-meridian-200 px-6 py-14 text-center dark:border-white/10">
          <DocumentCheckIcon class="h-10 w-10 text-meridian-400" />
          <h2 class="mt-3 font-medium text-meridian-800 dark:text-meridian-100">No contracts in this view</h2>
          <button type="button" @click="contractFilter = 'all'" class="mt-3 text-sm font-medium text-meridian-600 hover:underline">Show all contracts</button>
        </div>
      </section>

      <!-- Reports -->
      <section v-else>
        <div class="card-base divide-y divide-meridian-100 dark:divide-white/5">
          <div v-for="r in reports" :key="r.id" class="flex items-center justify-between p-5">
            <div class="flex items-center gap-3">
              <span class="flex h-10 w-10 items-center justify-center rounded-xl bg-meridian-50 dark:bg-white/5 text-meridian-600">
                <DocumentTextIcon class="h-5 w-5" />
              </span>
              <div>
                <p class="text-sm font-medium text-meridian-800 dark:text-meridian-100">{{ r.name }}</p>
                <p class="text-xs text-meridian-500">{{ r.period }} · {{ r.size }}</p>
              </div>
            </div>
            <div class="flex items-center gap-3">
              <BaseBadge tone="neutral">{{ r.type }}</BaseBadge>
              <button @click="download(r)" class="text-xs font-medium text-meridian-600 hover:text-pulse-500">Download</button>
            </div>
          </div>
        </div>
      </section>
    </template>

    <!-- PDF viewer -->
    <Teleport to="body">
      <div v-if="viewing" class="fixed inset-0 z-50 flex items-center justify-center p-4" role="dialog" aria-modal="true" :aria-label="viewing.title">
        <div class="absolute inset-0 bg-black/50" @click="closeViewer" />

        <div class="relative flex h-[88vh] w-full max-w-4xl flex-col overflow-hidden rounded-2xl border border-meridian-100 bg-white shadow-xl dark:border-white/10 dark:bg-meridian-900">
          <header class="flex items-center justify-between gap-3 border-b border-meridian-100 px-5 py-4 dark:border-white/10">
            <div class="min-w-0">
              <h2 class="truncate font-display text-base font-semibold text-meridian-900 dark:text-white">{{ viewing.title }}</h2>
              <p class="truncate text-xs text-meridian-500">{{ viewing.subtitle }}</p>
            </div>
            <div class="flex shrink-0 items-center gap-2">
              <a :href="viewing.file" target="_blank" rel="noopener noreferrer" :class="iconBtn" aria-label="Open in new tab">
                <ArrowTopRightOnSquareIcon class="h-4 w-4" />
              </a>
              <a :href="viewing.file" :download="`${viewing.title}.pdf`" :class="iconBtn" aria-label="Download PDF">
                <ArrowDownTrayIcon class="h-4 w-4" />
              </a>
              <button type="button" @click="closeViewer" :class="iconBtn" aria-label="Close">
                <XMarkIcon class="h-4 w-4" />
              </button>
            </div>
          </header>

          <iframe :src="viewing.file" :title="viewing.title" class="h-full w-full flex-1 bg-meridian-50 dark:bg-white/5" />
        </div>
      </div>
    </Teleport>
  </DashboardLayout>
</template>